---
doc_type: design-sketch
topic: Agent runtime — unified design for resume+caching and genealogy-based multi-agent handoff on Kafka
project: Greenwood
tags: [runtime, kafka, resume, prompt-caching, genealogy, lineage-dag, handoff, multi-agent, determinism, provenance, governance]
sources: [claude-api-skill, spark-to-fire, engineering]
created: 2026-06-30
updated: 2026-06-30
status: draft — the core unified design
summary: One stateless-recoverable runtime component per agent instance, over Kafka. Resume works because the prompt is a DETERMINISTIC projection of the event log (content-addressed cache hits from any host). The SAME determinism makes cross-agent handoff cacheable AND traceable — an agent is handed a canonical slice of the CONFIRMED lineage DAG (not the parent's raw transcript), which (a) is a stable shared cache prefix across sibling agents, (b) carries provenance so outcomes trace back to root claims, and (c) gates error cascades at the agent boundary (Spark-to-Fire applied to handoff).
---

# Agent runtime: resume + caching + genealogy handoff (unified)

The insight tying it all together: **resume and multi-agent handoff are the same
problem** — reconstruct an agent's trusted context from the event log, deterministically,
so it hits the content-addressed LLM cache. Resume reconstructs *one agent's own*
context after a crash; handoff constructs *a new agent's* context from a parent's. Both
are deterministic projections of the same lineage DAG. Build the projection once and you
get resume, cheap fan-out, and traceable handoff for free.

## 1. The runtime component
One process = one running agent instance. It is **stateless-recoverable**: it holds no
durable state that isn't derivable from Kafka, so any runtime instance on any pod can
pick up any agent. Its loop:
1. Consume the agent's ordered event stream (its partition) + the confirmed-lineage it
   depends on.
2. Assemble the prompt deterministically (see §3).
3. Call the LLM (`claude-opus-4-8`), stream output deltas back to Kafka.
4. Execute tool calls, emit results as events.
5. Emit the turn's context manifest (mechanical provenance — D7); Grieve's bus-side
   decomposer turns its output into atomic claims and the triage classifier screens them.
6. On handoff/spawn, emit a handoff event referencing confirmed claim-ids (§5).

## 2. The event envelope (the atom)
Kafka record `value`, Protobuf + Schema Registry (versioned — fixes the garden's
no-version regret). Core fields:
- `event_id` (content-hash → replay-idempotent), `schema_version`, `ts`
- `agent_session_id`, `lineage_root` (partition key — see §3)
- `kind`: `thought|tool_call|tool_result|claim|handoff|snapshot|stream_delta|control`
- for claims: `claim_id`, `claim_kind` (factuality|faithfulness), `content`,
  `evidence_ref`, **provenance edges** `derived_from[] / supports[] / contradicts[]`,
  `trust_state` (unverified|confirmed|rejected|quarantined — set by governance)
- for handoffs: `from_agent`, `to_agent`, `task_spec`, `context_claims[]` (the
  confirmed claim-ids being handed)
- optional `provider_session` (null for Anthropic — stateless; carries an
  OpenAI-Responses id if a backend is server-stateful)

## 3. Kafka topics & the determinism spine
- **`raw`** — every event, immutable. **Keyed by `lineage_root`** so a whole
  interaction branch is totally ordered in one partition (Chattermax keyed by random
  UUID and lost ordering — we do NOT). An agent's own events are a contiguous,
  ordered sub-stream within its lineage.
- **`confirmed`** — log-compacted; latest `trust_state` per `claim_id`. This is the
  **trusted context** and the **cacheable prefix source**. Governance is the only writer.
- **`control`** — spawn/handoff/freeze/fork/rewind/abandon lifecycle events.
- **`snapshots`** — periodic serialized agent state + the offset it reflects.
- **`dlq`**.

**Determinism is the load-bearing invariant** (it's what makes both resume and handoff
cache-friendly). Everything that enters a cached prefix must be a pure function of the
log: frozen system prompt, deterministically-sorted tool defs, canonical (sorted-key)
serialization, no timestamps/uuids/pod-ids/nonces before the last `cache_control`
breakpoint. Break this and `cache_read_input_tokens` silently goes to 0.

## 4. Resume (one agent, after reschedule)  [detail in topic 06]
New pod acquires the agent (Kafka consumer-group rebalance for small fleets;
claim-on-`control` queue to scale past partition count) → loads latest `snapshots`
entry → replays `raw` after that offset → **rebuilds the messages array
byte-identically** → re-issues the next LLM call with the same breakpoints → **cache
hit from a different host** (Anthropic cache is content-addressed, not session-bound;
shared across the org). TTL is per-agent policy — 5-min default; `ttl:"1h"` only for 5–60-min turn cadence (the write premium exceeds the crash-resume payout otherwise; topic 06). In-flight
call at crash → re-issue (prefix cached, generation redone); stream deltas into `raw`
to salvage partials. `event_id` = content hash → replay never double-emits.

## 5. Genealogy handoff (across agents) — the new part
When agent A hands work to B (delegate, spawn sub-agent, navigator→implementer):

**A does NOT dump its transcript into B.** Instead:
1. A emits a `handoff` event: `task_spec` + `context_claims[]` = a set of **confirmed**
   claim-ids (a curated slice of the lineage DAG).
2. The runtime spawns B. B's prompt is assembled in cache-friendly layers:
   ```
   [ B-role system prompt ]                         ← breakpoint 1: shared by ALL B-role agents
   [ canonical serialization of the confirmed        ← breakpoint 2: shared by all agents
     lineage slice A handed (claims in id order) ]     handed THIS slice (e.g. sibling
                                                        implementers on one feature)
   [ B's task instruction ]                          ← per-B
   ... B's own turns append here ...
   ```
3. B's outputs are emitted with `derived_from` pointing at the handed claim-ids —
   provenance crosses the A→B boundary explicitly.

This buys three things at once:

**(a) Readable, governed context (NOT primarily a caching play — see D2).** Caching is
*not* a design driver at handoff: a fresh B has a cold cache, so the payload is a first
write, not a read. So the payload is built for *quality + safety*, not byte-stability.
It is **graduated / progressive disclosure** (D2): Tier 0 = a verified LLM
selection+synthesis briefing (carries `derived_from` edges; entail-checked against its
sources before crossing); Tier 1 = expand to the underlying confirmed claims; Tier 2 =
expand to evidence. The synthesis is still **logged as an event** — for provenance and
for B's *own* later resume (replay reads the logged slice; never re-run the LLM on
resume). (Incidental: if two siblings happen to get an identical logged slice they
*could* share cache, but we don't engineer for it.)

**(b) Traceable handoff / root-cause attribution.** Because B's claims are
`derived_from` A's confirmed claims (which are `derived_from` their evidence), any B
outcome walks back through the DAG to root claims and sources across agent boundaries.
This is the garden's missing provenance (its event stream was temporal-only; topic 01)
made first-class and cross-agent.

**(c) Error-cascade gating at the boundary — Spark-to-Fire applied to handoff.** B is
handed **confirmed** claims only; A's unverified/rejected/quarantined claims never enter
B's trusted context. The handoff is a governance checkpoint: false-consensus can't
propagate A→B because only screened claims cross. And when a handed claim is later
flipped to `rejected`, you query the DAG for every agent whose work is `derived_from`
it (∪ the mechanical context-manifest envelope — D7) and
**rewind/re-verify just those branches** — targeted multi-agent rollback, not a global
redo. Scope honesty: rewind = compensating event + re-derive of *derived state*, never
deletion (topic 02) — it repairs belief; external tool effects need the effect gate +
compensation story (topic 08 §5a), and re-running a branch is stochastic regeneration
at inference cost.

## 6. How the three compose (the payoff)
- **Determinism** (§3) is the single mechanism. It makes an agent's context a pure
  function of the log.
- **Resume** = reconstruct *my own* context deterministically → cache hit.
- **Handoff** = construct *your* context from a deterministic slice of *my* confirmed
  lineage → cache hit (and shareable across siblings).
- **Genealogy** = the provenance edges that make that slice selectable, traceable, and
  governable.
So the lineage DAG is simultaneously: the resume substrate, the handoff payload, the
cache-prefix source, and the audit/rollback structure. One structure, four jobs.

## 7. What one runtime tick looks like (sketch)
```
loop:
  events   = consume(raw, key=my.lineage_root, from=my.committed_offset)
  ctx      = confirmed_slice(my.context_claims) ++ my.turns(events)   # deterministic
  prompt   = layer(system[bp1], ctx[bp2], task, my.turns)             # cache_control at bp1,bp2
  resp     = llm.call(prompt, ttl=policy.ttl)                          # cache hit on bp1/bp2
  emit(raw, stream_deltas(resp))                                      # salvageable partials
  for call in resp.tool_calls: emit(raw, tool_result(exec(call)))
  claims   = decompose(resp)                                          # atomic claims
  emit(raw, claims)                                                   # governance screens → confirmed
  if resp.wants_handoff: emit(control, handoff(to=B, context_claims=select_confirmed()))
  if turn_count % N == 0: emit(snapshots, serialize(ctx), offset)
  commit_offset()
```

## 8. Open questions
- ~~Claim decomposition owner/cost~~ **decided (D7, 2026-07-02)**: bus-side decomposer +
  mechanical context manifests; cost bounded by an always-on triage classifier
  (FActScore/NLI style per the paper) that reserves LLM verification for
  yellow/hub/boundary claims.
- **What granularity crosses a handoff** — atomic claims, or claim-clusters/"findings"?
  Too fine = noisy prompt; too coarse = loses trace resolution.
- **Governance timing** — inline screen before `confirmed` write (latency) vs. async
  quarantine-until-confirmed (fits Kafka; consumers must honor `confirmed` as the trust
  boundary and not read `raw` for trusted context). Topic 02.
- **Snapshot cadence** N vs. replay cost; whether snapshot inlines the confirmed slice
  (stable bytes, cheap rebuild) or points to it.
- **Cross-lineage handoff** — if B's work spans two lineage_roots, its ordering/partition
  story needs care (a handoff can join branches). Consequence (red-team, 2026-07-02):
  because edges cross lineage_roots, a per-lineage-partitioned projector cannot answer
  `descendants()` locally — Rootlines needs a **global read store** (DB-backed
  projection), and its risk/centrality signals *lag the log*: a claim becomes a hub only
  after adoption, so hub-based selective verification structurally trails adoption. The
  boundary gate is the compensating control (staleness at a gate fails conservative);
  record "hub detection lags adoption" as a known limit.
