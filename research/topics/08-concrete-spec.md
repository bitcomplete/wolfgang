---
doc_type: technical-spec
topic: Greenwood concrete spec — event envelope, Kafka topics, projections/folds, Grieve policy cascade, messaging
project: Greenwood
tags: [spec, protobuf, envelope, kafka, topics, partitioning, projection, fold, verifier, messaging, policy, control, buildable]
sources: [decisions, topics-01-07, claude-api-skill, spark-to-fire]
created: 2026-06-30
updated: 2026-06-30
status: draft — the buildable layer; tickets reference this
summary: The concrete artifacts an implementer needs — the Protobuf event envelope + payload variants, the Kafka topic/partition layout, the fold rules for the confirmed/lineage/agent-state projections, the Grieve per-message policy cascade, and messaging/delivery + the spawn-message pattern. Encodes P0/D1/D2/D3/D7/D8.
---

# Greenwood — concrete spec (buildable layer)

Encodes the decisions log (P0 event-sourcing-everywhere; D1 agent-proposes/verifier-
confirms; D2 graduated tiering (now the spawn-message pattern); D3 async gating, applied
per message; D7 bus-side decomposition; D8 messaging + per-message policy). This is the layer tickets
point at. Protobuf shapes are illustrative (names/field-numbers TBD in the schema ticket).

## 1. Event envelope (Protobuf + Schema Registry)
```proto
message Event {
  string   event_id        = 1;  // content hash — see canonical-hash spec below
  uint32   schema_version  = 2;
  int64    ts              = 3;  // wall clock, NEVER inside a cached prefix (topic 06), NEVER in the hash
  string   producer        = 4;  // agent/verifier/runtime id
  string   agent_session_id= 5;
  string   lineage_root     = 6;  // partition key (see §2)
  uint64   session_epoch   = 7;  // producer-fencing epoch (see §2a) — folds reject stale epochs
  oneof payload {
    Thought          thought          = 10;
    ToolCall         tool_call        = 11;
    ToolResult       tool_result      = 12;   // = EVIDENCE, not a claim (topic 03)
    Claim            claim            = 13;
    TrustTransition  trust_transition = 14;
    Handoff          handoff          = 15;   // legacy alias — new code uses Message{SPAWN_BRIEF} (D8)
    StreamDelta      stream_delta     = 16;
    Snapshot         snapshot         = 17;
    Control          control          = 18;
    ContextManifest  context_manifest = 19;   // D7: mechanical provenance
    TriageScore      triage_score     = 20;   // D7: classifier screen
    Message          message          = 21;   // D8: the ONLY inter-agent primitive
    PolicyDecision   policy_decision  = 22;   // D8: per-message governance ruling
  }
}
message Message {                     // D8: agents communicate ONLY by messages
  string msg_id = 1;
  string to = 2;                      // agent_session_id or room id
  enum MsgType { CHAT=0; SPAWN_BRIEF=1; RESULT=2; ACK=3; STATUS=4; TOOL_FORWARD=5; } MsgType msg_type = 6;
  string body = 3;
  repeated string refs = 4;           // claim/evidence ids referenced
  string in_reply_to = 5;
}
message PolicyDecision {              // D8: every governance ruling is an event (P0)
  string msg_id = 1;
  enum Route { DELIVER=0; SCREEN=1; VERIFY=2; BLOCK=3; SAMPLED_AUDIT=4; } Route route = 2;
  string rule_id = 3;                 // static rule that matched, or "encoder"
  string policy_version = 4;          // static policy config is versioned via events
}
message Claim {                       // D7: emitted by the BUS-SIDE DECOMPOSER, lands unverified
  string claim_id = 1;
  enum Kind { FACTUALITY = 0; FAITHFULNESS = 1; } Kind kind = 2;
  string content = 3;
  string evidence_ref = 4;            // tool_result/source it should be checked against
  repeated string derived_from = 5;   // inferred by the decomposer + turn's ContextManifest (D7)
  repeated string supports = 6;
  repeated string contradicts = 7;
  repeated string tags = 8;           // for deterministic grouping in spawn briefs (D2 pattern)
  string source_turn_id = 9;
  repeated string agent_hint_edges = 10; // optional agent-declared edges — untrusted hints only
}
message ContextManifest {             // D7: MECHANICAL provenance, emitted by the RUNTIME per turn
  string turn_id = 1;
  repeated string claim_ids_in_context = 2;  // confirmed claims present in the prompt
  repeated string evidence_refs_in_context = 3;
}                                     // un-gameable: the runtime built the prompt, so it knows
message TriageScore {                 // D7: classifier screen, always-on, milliseconds
  string claim_id = 1;
  enum Verdict { GREEN = 0; RED = 1; YELLOW = 2; }  // entailed / contradicts / uncertain
  Verdict verdict = 2; float score = 3; string classifier_version = 4;
}
message TrustTransition {             // D1: emitted by the VERIFIER; P0: trust is event-sourced
  string claim_id = 1;
  enum State { UNVERIFIED=0; CONFIRMED=1; REJECTED=2; QUARANTINED=3; } State to_state = 2;
  string verifier_id = 3;
  repeated string verdict_derived_from = 4;  // the claim + evidence checked → verifier auditable
  string rationale_ref = 5;
  float  risk_score = 6;              // spectral/hub signal (Spark-to-Fire)
}
message Handoff {                     // D2/D3 — superseded by Message{SPAWN_BRIEF} (D8); kept for schema history
  string from_agent = 1; string to_agent = 2;
  string task_spec = 3;
  repeated string seed_claim_ids = 4; // A's selected conclusion claims
  string synthesis_claim_id = 5;      // Tier-0 verified synthesis (a Claim, derived_from the slice)
  ExpansionPolicy expansion = 6;      // depth bound, lazy vs eager
}
message ToolCall {
  string call_id = 1; string tool = 2; bytes args = 3;
  enum EffectClass { PURE=0; IDEMPOTENT=1; REVERSIBLE=2; IRREVERSIBLE=3; }
  EffectClass effect_class = 4;       // declared by the tool REGISTRY, not the agent
  string compensation_ref = 5;        // for REVERSIBLE: the registered undo handler
}
message Control { enum Op { SPAWN=0; FORK=1; REWIND=2; ABANDON=3; FREEZE=4; RESUME=5;
    ACQUIRE=6;            // bump session_epoch on agent acquisition (producer fencing)
    SERIALIZATION_EPOCH=7; // harness/graft/schema upgrade: force snapshot, declare cache cold
  }
  Op op = 1; string target = 2; map<string,string> params = 3; }
message Snapshot { string agent_session_id=1; int64 offset=2; string state_ref=3; repeated int32 breakpoints=4; }
message StreamDelta { string stream_id=1; uint64 seq=2; bool complete=3; string text=4; }
```

**Canonical-hash spec (`event_id`).** `event_id = sha256(schema_version ‖ lineage_root ‖
partition-offset-position ‖ canonical(payload))` where `canonical()` is a pinned,
per-`schema_version` field-ordered serialization — **not** raw Protobuf bytes (Protobuf
encoding is not canonical across libraries/languages). `ts`, `producer`, and
`session_epoch` are **excluded** — re-emission and re-acquisition must not change the
hash, or replay idempotency breaks. Conformance suite ships cross-language hash test
vectors. On any change to prompt assembly or `canonical()` (harness upgrade, graft
upgrade, schema migration): emit `Control{SERIALIZATION_EPOCH}` — force a snapshot,
declare the prompt cache cold, and never mix assembly versions inside one prefix.

## 2. Kafka topics & partitioning
| topic | key | retention | purpose |
|---|---|---|---|
| `agent-bus.raw` | `lineage_root` | tiered (hot→S3, KIP-405) | immutable spine; a branch is totally ordered in one partition |
| `agent-bus.confirmed` | `claim_id` | compacted | **projection**: latest trust state per claim (trusted context) |
| `agent-bus.lineage` | `claim_id` | compacted | **projection**: DAG adjacency (edges) for traversal/rollback |
| `agent-bus.control` | `agent_session_id` | tiered | spawn/fork/rewind/abandon/freeze/resume |
| `agent-bus.snapshots` | `agent_session_id` | compacted (keep latest) | resume checkpoints |
| `agent-bus.dlq` | — | — | poison |

- **Key by `lineage_root`, NOT random UUID** (Chattermax's deferral lost ordering — D-ref topic 01/05). A claim's trust-transitions land in the same partition as the claim → ordered folds.
- `confirmed`/`lineage` are **projections** (P0) built by a projector consumer folding `raw` — never authoritative, always rebuildable by replay.
- **Fleet-scale caveat (measured, 2026-07-02):** `claim_id` is a *monotone* keyspace —
  each key is written ~2–3× (unverified→confirmed) then never again, so latest-per-key
  compaction converges to the whole claim set (~25M new keys/hr at 500k agents; ~112 TB
  "compacted" over a year; WarpStream's 3M-keys/partition compaction budget exhausted in
  hours). Compaction is fine at small fleets and for genuinely bounded keyspaces
  (`snapshots` by `agent_session_id`); for claim state at fleet scale add **TTL after
  terminal state** (a confirmed/rejected claim is immutable — archive to the raw log and
  drop from the compacted view) or make `confirmed` a DB-backed projection.

## 2a. Producer fencing (zombie agents)
Consumer-group membership fences *consumption*, not *production*: a network-partitioned
"dead" pod can keep producing, and since LLM output is stochastic, its events won't hash
equal to the new owner's — two divergent histories would interleave under one
`agent_session_id`. Fence with the **`session_epoch`**: acquisition emits
`Control{ACQUIRE}` which bumps the epoch; every event carries it; **folds drop events
whose epoch is stale**. (Alternative: Kafka transactional producer with per-session
`transactional.id` for broker-enforced fencing.) Claim-queue acquisition gets an expiry;
"zombie producer" is a required scenario in the resume tests and Graft conformance suite.

## 2b. Read access (closing topic 02's open question)
"Consumers read `confirmed`, not `raw`" is enforced by **ACL/capability, not convention**:
agent-role principals get no read on `raw`/`lineage`; only the projector, Grieve, the
gate, and ops tooling do. An agent's trusted context is *constructed for it* by the
runtime from `confirmed`.

## 3. Fold rules (projections are derived state — P0)
- **confirmed[claim_id]** = the `Claim` with `trust_state` = `to_state` of its latest
  `TrustTransition` (by partition order). `REJECTED`/`QUARANTINED` ⇒ excluded from any
  trusted context / spawn-brief slice.
- **lineage DAG** = nodes: claims; edges: `derived_from`/`supports`/`contradicts`
  (decomposer-inferred) + the turn-level **ContextManifest** edges (mechanical — D7).
  Key query: `descendants(claim_id)` = rollback blast radius (who relied on X). Rewind
  scope = fine-grained descendants ∪ the manifest envelope (every turn that had the
  rejected claim in context, and everything derived from those turns) — the manifest is
  the honest, un-gameable bound.
- **agent state (resume)** = fold of `raw` for `agent_session_id` from the last
  `Snapshot.offset` forward → the messages array (topic 06). Must be a **pure function
  of the log** (canonical serialization; no ts/uuid/pod-id in the cached prefix).

## 4. Grieve pipeline (separate process — D1/D7/D8, topic 02)
Governance runs **on the message path only** (D8), as a per-message policy cascade —
cheapest stage first; every ruling logged as a `PolicyDecision` event:
1. **Static rules (no ML, free).** Declarative config matched on `msg_type` + attributes:
   `ACK`/`STATUS`/`TOOL_FORWARD` → DELIVER untouched; `SPAWN_BRIEF` and anything
   addressed to a **hub role** → SCREEN always; effect-gate premises → VERIFY always.
   Policy config changes are events (versioned, auditable, replayable).
2. **Encoder triage** (self-hosted NLI-small, ~ms, ~$0.02/M tok) for messages the rules
   don't decide: green → DELIVER, with a **sampled-audit** rate (greens occasionally
   decomposed anyway) so the false-green rate is a measured SLO; else →
3. **Decomposer (D7, message-scoped):** splits the message into atomic `Claim`s
   (FActScore-style; small/cheap model), inferring `derived_from` from content + the
   sender's `ContextManifest`; claims screened tri-state against confirmed context →
   `TriageScore{GREEN|RED|YELLOW}`. RED ⇒ **block + feedback package** to sender;
   YELLOW →
4. **Verifier (LLM, selective):** the paper's Balanced policy. Others stay `UNVERIFIED`
   (excluded from trusted context, never forwarded-with-tag).

**Tuning loop (D7, unchanged):** `TrustTransition` verdicts + sampled-audit divergences
are accumulating labeled data — periodically fine-tune the encoder (and distill the
decomposer) on them; every new model version is gated through Coldframe (D4) before
promotion, like any harness change. Trust states are still assigned only by the
verifier. Intra-agent traffic is never eagerly decomposed; `raw` is immutable, so
**retroactive decomposition** (the paper's offline mode) covers forensics.
- **Factuality** check: against `evidence_ref` / tools / world.
- **Faithfulness** check: entailment against the evidence bundle the producer was given
  (unsupported/contradictory ⇒ `REJECTED`).
- Emits `TrustTransition` (its own event; `verdict_derived_from` = claim + evidence).
- Circuit breaker: cap retries `K`; persistent-unresolved ⇒ `QUARANTINED` (high-risk
  tagged, excluded from trusted context).
- The verifier is itself an LLM step → its verdicts are auditable events (P0 "back up").

## 5. Messaging & delivery (D8 — replaces the handoff protocol)
1. A emits `Message{to, msg_type, body, refs}` — to an agent or a room. Messages ride
   `raw` like every event (keyed by `lineage_root`; a room is a lineage-ish addressing
   convention, not a broker feature).
2. The **Grieve cascade (§4)** rules on it: DELIVER / SCREEN / VERIFY / BLOCK, logged as
   a `PolicyDecision`. Screen-stage reads are **offset-aligned**: wait until `confirmed`
   and the lineage projection have both passed the message's offset, else the screen can
   see a claim CONFIRMED whose ancestry edges haven't landed.
3. **Delivery = projection**: a recipient's inbox is a fold of messages whose ruling
   permits delivery. An agent's trusted context only ever accumulates screened content —
   D3's guarantee, message-by-message. Blocked messages return to the sender as a
   feedback package (rejected claims + conflict evidence + rewrite directive).
4. If a delivered claim later flips `REJECTED`, emit `Control{REWIND}` scoped by the
   recipient set (who consumed it — mechanical, from manifests): belief repair +
   quarantined descendants; REVERSIBLE external effects run registered compensation,
   IRREVERSIBLE ones surface for human remediation. Re-running a branch is stochastic
   regeneration at inference cost, not a deterministic re-fold.

**The spawn-message pattern (formerly "handoff" — D2's machinery as a library):**
compose `Message{msg_type: SPAWN_BRIEF}` from the confirmed provenance subgraph (seeds +
ancestors, depth-bounded, confirmed-only) → **Tier-0 synthesis** (a `Claim`,
`derived_from` the slice, entail-checked against it — the entailment check is a
stochastic filter with a monitored false-pass rate; run two independent checkers for
high-risk spawns) → `Control{SPAWN}`; B's prompt = [B-role system] + [brief] + [task],
Tiers 1/2 via lazy expansion. Static policy routes every `SPAWN_BRIEF` to SCREEN, so the
old boundary gate falls out of the ordinary message path rather than existing as a
separate mechanism.

## 5a. Effect gate (symmetric to the message screen)
Trust must govern **actions**, not just messages: before executing a `ToolCall` with
`effect_class = IRREVERSIBLE`, the runtime requires the claims it's premised on (the
turn's `ContextManifest` set — mechanical, D7) to be **confirmed** — or escalates for
human approval. `REVERSIBLE` requires a registered `compensation_ref`. `PURE`/`IDEMPOTENT`
execute freely. This is the rollback story's other half: the cheaper the gate lets an
unverified branch act on the world, the less "rewind" can ever mean.

## 6. Runtime loop (ties it together — topic 07 §7)
consume(raw@lineage_root) → fold state → assemble prompt (cache_control bps) →
LLM (opus-4-8, ttl per cadence policy) → emit stream_deltas → exec tools (effect gate §5a) → emit tool_results →
emit ContextManifest (D7; the decomposer + triage classifier run bus-side, not in the
agent loop) → send/receive Messages via the delivery projection (§5) → snapshot every N → commit.

## 7. Cross-cutting (garden's unsolved ops problems — design targets)
Exec/tool sandboxing; rate limiting (runaway agents); cost control (per-agent token
budgets — Anthropic `task_budget`); crash/failure recovery (resume covers the happy
path; need detection); multi-agent test harness (simulation). Tracked as their own
work area.
