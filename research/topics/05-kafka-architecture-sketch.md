---
doc_type: design-sketch
topic: Greenwood on Kafka — architecture sketch (Kafka substrate, garden as inspiration)
project: Greenwood
tags: [kafka, architecture, event-sourcing, lineage-dag, governance, schema-registry, kafka-streams, topics, partitioning]
sources: [spark-to-fire, engineering]
created: 2026-06-30
updated: 2026-06-30
status: draft — Terra's decision: build on Kafka, NOT bound to Chattermax's other choices
summary: First architecture sketch for Greenwood on Kafka. Kafka's log IS both transport and immutable event store (collapses Chattermax's XMPP/Kafka split). Atomic-claim envelope via Schema Registry; lineage DAG + governance as Kafka Streams projections/processors; pruning = compensating events (never deletion). Flags divergences from Chattermax and open decisions.
---

# Greenwood on Kafka — architecture sketch

**Decision (Terra, 2026-06-30):** build on top of Kafka. Chattermax / the garden are
*inspiration*, not constraints. Below I take the good ideas (event sourcing,
transport-not-merge, projections, atomic-claim governance from *Spark to Fire*) and
realize them Kafka-natively, flagging where I diverge and why.

## 0. What Kafka gives us for free (vs. garden's hand-rolled event sourcing)
- **Append-only, immutable, ordered-within-partition log** = event sourcing native.
  The garden's "state is DERIVED not STORED / immutable log is source of truth" is
  just how Kafka works.
- **Log IS transport.** Consumers tail in real time → no separate messaging layer.
  → **Divergence from Chattermax:** drop XMPP entirely; Kafka does real-time + durable.
- **Tiered storage (KIP-405)** = the Kafka→S3 cold-storage tiering the garden
  designed by hand (the prior system). Hot local + cold S3, transparent.
- **Schema Registry** = solves the garden's "no version field, no migration plan"
  regret directly (schema evolution / compatibility built in).
- **Consumer groups + offsets** = replay, time-travel, and "resume from point"
  primitives, per-consumer.
- **Log compaction** = materialized "latest state per key" topics (e.g. confirmed
  claim state) without an external DB. *Fleet-scale caveat:* the claim keyspace is
  monotone (keys written ~2–3× then never again), so compaction converges to the whole
  claim set at large fleets — claim state then needs TTL-after-terminal-state or a
  DB-backed projection (measured analysis: topic 08 §2, D6).
- **Caveat — tiering and compaction don't mix on vanilla Kafka:** KIP-405 explicitly
  does **not** support compacted topics. So the raw log tiers to S3, but compacted
  topics (`confirmed`, lineage projections) must stay on local replicated disk. They're
  small (latest-per-key), so this is workable — and S3-native engines like AutoMQ
  don't have the restriction at all (compaction runs against S3-backed storage; see D6).

## 1. The event envelope (the atom)
Kafka record `value` = an **atomic-claim event** (see topic 03). Protobuf +
Confluent Schema Registry (versioned, backward-compatible). Core fields:
- `event_id` (uuid), `schema_version`
- `claim_id`, `kind` (factuality|faithfulness|action|thought|…)
- `content`, `evidence_ref` (the bundle it was conditioned on — paper's `Eᵢ(t)`)
- **provenance:** `derived_from[]`, `supports[]`, `contradicts[]`, `revises[]`
- `trust_state` (unverified|confirmed|rejected|quarantined) — usually set by the
  governance projection, not the producer
- `producer` (agent id), `branch_id` (agent-trajectory branch), `ts`
- risk annotations: `source_centrality`, `risk_score` (populated by projection)
- Record `key` → partitioning (see §2). Record `headers` → routing/trace ids.

**Coarse messages coexist:** an agent turn may emit one "message" event plus N
atomic-claim events that reference it (`derived_from` the turn). Cheap paths can skip
decomposition (paper's Low-Intervention policy).

## 2. Topics & partitioning (ordering is the crux)
- **`agent-bus.raw`** — every claim/message event as produced (the immutable spine).
- **`agent-bus.confirmed`** — compacted; latest `trust_state` per `claim_id` (the
  "trusted context" the paper's confirmed-lineage represents).
- **`agent-bus.control`** — lifecycle & governance actions (spawn, freeze, fork,
  rewind, quarantine, compensating events).
- **`agent-bus.dlq`** — poison/failed events.
- **Partitioning / ordering:** key by **`lineage_root`** (or `conversation_id`) so an
  entire interaction branch lands in ONE partition → total order within a branch,
  parallelism across branches. (Keying by agent-id instead would scatter a branch
  across partitions and lose causal order — avoid.) This is the Kafka answer to
  "rooms need ordered history" without MUC.
- **Rooms?** Optional. If we want Chattermax-style project/feature rooms, a "room" is
  just a `lineage_root`/topic-prefix convention, not a broker feature. Presence (who's
  "in" a room) would be a separate compacted state topic if needed. **Default: skip
  presence; agents are consumers, not avatars.**

## 3. The two coupled trees (topics 01 & 04) as projections
Both are **Kafka Streams / consumer-side projections**, NOT broker features — honors
"transport stays dumb; understanding is derived" (topic 02).
- **Lineage DAG projection** — consumes `raw`, builds the genealogy graph
  (`supports`/`contradicts`/`derived_from` edges, confirmed/unverified state) in a
  state store (RocksDB) and/or an external graph DB. Computes `risk_score` &
  `source_centrality` (paper's spectral/hub insight). This is the read model for
  monitoring + pruning.
- **Agent-trajectory tree** — consumes `control`; tracks branches with Gardener-style
  `Forked`/`Rewound`/`Promoted`/`Abandoned` events. Maps a rejected claim-subtree →
  the agent branch(es) that produced/relied on it.

## 4. Governance coordinator (the paper, on Kafka)
A **stream-processing app** (not in the broker; a Navigator-style coordinator role,
topic 02). Pipeline mirrors *Spark to Fire* §VI:
1. **Decompose** incoming turns → atomic claims (optional LLM pass; gate by risk).
2. **Screen** each claim against the lineage DAG / `confirmed` topic → 🟢 entailed /
   🔴 contradicts / 🟡 unknown.
3. **Policy-route 🟡** (Speed / Balanced=verify hub-role claims / Strict) → verify via
   retrieval + NLI → resolve.
4. **Actuate:** 🟢 → publish to `confirmed`. 🔴 → publish a **compensating/quarantine
   event** to `control` + feedback package to producer; **never delete** from `raw`
   (immutability, topic 02). Circuit breaker: cap retries `K`, isolate persistent
   offenders.
- **Pruning = "never promote to confirmed" + compensating event + re-derive
  downstream state.** The faulty branch stays in `raw` for audit; it's excluded from
  `confirmed`. Consumers of trusted context read `confirmed`, not `raw`.
- Key finding to honor: **detection alone fails; the block/rollback/isolation step is
  what contains cascades** (paper ablation). So the actuation step is mandatory, not
  optional monitoring.

## 5. Agents & spawning (exec-hook idea, Kafka-native)
- Agents are **consumers** (in consumer groups) + **producers**. Real-time = tailing.
- Chattermax's exec-hook = a **spawner service** consuming `raw`/`control`, matching
  filters (e.g. a `work_available` claim), and launching agent processes. Kafka gives
  the at-least-once delivery + offset tracking the garden hand-built.
- **Failure recovery** (garden's open question #7): a spawned agent's progress is its
  emitted events; a crash → its branch is incomplete → coordinator can `Rewind` from
  last confirmed offset or `Abandon` the branch. This is the garden gap Greenwood
  closes *because* the log makes progress reconstructible.

## 6. What we deliberately DROP from Chattermax
- **XMPP / MUC / presence / 12 XMPP message types / Git XEP transport** — replaced by
  Kafka topics + a Protobuf claim schema. (Git can still be the merge substrate;
  the bus just carries `code_change` claims + provenance.)
- **Disk-scanning freeze/thaw bridge** — unnecessary; freeze = an offset/range +
  compacted snapshot; replay is native. (Avoids the garden's #1 regret.)

## 7. Open decisions (need Terra / more discovery)
- **Schema format:** Protobuf+Schema Registry (recommended) vs. Avro vs. JSON.
- **Lineage store:** Kafka Streams state store (RocksDB) only, vs. external graph DB
  (Neo4j/Memgraph) for rich `supports/contradicts` queries. Likely both (RocksDB hot,
  graph DB for analysis).
- **Do we need rooms/presence at all,** or is topic + `lineage_root` keying enough?
- **Synchronous inline governance (per-hop latency, paper-style) vs. async
  quarantine-until-confirmed gate.** Async fits Kafka better; needs consumers to honor
  the `confirmed` topic as the trust boundary.
- **Claim decomposition owner:** producing agent (self-declared) vs. a bus-side
  decomposition service (trust/cost tradeoff).
- **Language/stack:** garden leans Rust; Kafka Streams is JVM-first (Rust needs
  rdkafka + hand-rolled stream logic, or a JVM/Kotlin streams app). Real decision.
