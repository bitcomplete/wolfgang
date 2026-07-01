---
doc_type: technical-spec
topic: Greenwood concrete spec — event envelope, Kafka topics, projections/folds, verifier, handoff protocol
project: Greenwood
tags: [spec, protobuf, envelope, kafka, topics, partitioning, projection, fold, verifier, handoff, control, buildable]
sources: [decisions, topics-01-07, claude-api-skill, spark-to-fire]
created: 2026-06-30
updated: 2026-06-30
status: draft — the buildable layer; tickets reference this
summary: The concrete artifacts an implementer needs — the Protobuf event envelope + payload variants, the Kafka topic/partition layout, the fold rules for the confirmed/lineage/agent-state projections, the verifier contract, and the handoff protocol + boundary gate. Encodes P0/D1/D2/D3.
---

# Greenwood — concrete spec (buildable layer)

Encodes the decisions log (P0 event-sourcing-everywhere; D1 agent-proposes/verifier-
confirms; D2 graduated handoff; D3 async-with-boundary-gate). This is the layer tickets
point at. Protobuf shapes are illustrative (names/field-numbers TBD in the schema ticket).

## 1. Event envelope (Protobuf + Schema Registry)
```proto
message Event {
  string   event_id        = 1;  // content hash of (payload + causal position) → replay-idempotent
  uint32   schema_version  = 2;
  int64    ts              = 3;  // wall clock, NEVER inside a cached prefix (topic 06)
  string   producer        = 4;  // agent/verifier/runtime id
  string   agent_session_id= 5;
  string   lineage_root     = 6;  // partition key (see §2)
  oneof payload {
    Thought          thought          = 10;
    ToolCall         tool_call        = 11;
    ToolResult       tool_result      = 12;   // = EVIDENCE, not a claim (topic 03)
    Claim            claim            = 13;
    TrustTransition  trust_transition = 14;
    Handoff          handoff          = 15;
    StreamDelta      stream_delta     = 16;
    Snapshot         snapshot         = 17;
    Control          control          = 18;
  }
}
message Claim {                       // D1: emitted by the AGENT, lands unverified
  string claim_id = 1;
  enum Kind { FACTUALITY = 0; FAITHFULNESS = 1; } Kind kind = 2;
  string content = 3;
  string evidence_ref = 4;            // tool_result/source it should be checked against
  repeated string derived_from = 5;   // provenance (agent knows best — D1)
  repeated string supports = 6;
  repeated string contradicts = 7;
  repeated string tags = 8;           // for deterministic grouping in handoff (D2)
  string source_turn_id = 9;
}
message TrustTransition {             // D1: emitted by the VERIFIER; P0: trust is event-sourced
  string claim_id = 1;
  enum State { UNVERIFIED=0; CONFIRMED=1; REJECTED=2; QUARANTINED=3; } State to_state = 2;
  string verifier_id = 3;
  repeated string verdict_derived_from = 4;  // the claim + evidence checked → verifier auditable
  string rationale_ref = 5;
  float  risk_score = 6;              // spectral/hub signal (Spark-to-Fire)
}
message Handoff {                     // D2/D3
  string from_agent = 1; string to_agent = 2;
  string task_spec = 3;
  repeated string seed_claim_ids = 4; // A's selected conclusion claims
  string synthesis_claim_id = 5;      // Tier-0 verified synthesis (a Claim, derived_from the slice)
  ExpansionPolicy expansion = 6;      // depth bound, lazy vs eager
}
message Control { enum Op { SPAWN=0; FORK=1; REWIND=2; ABANDON=3; FREEZE=4; RESUME=5; }
  Op op = 1; string target = 2; map<string,string> params = 3; }
message Snapshot { string agent_session_id=1; int64 offset=2; string state_ref=3; repeated int32 breakpoints=4; }
message StreamDelta { string stream_id=1; uint64 seq=2; bool complete=3; string text=4; }
```

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

## 3. Fold rules (projections are derived state — P0)
- **confirmed[claim_id]** = the `Claim` with `trust_state` = `to_state` of its latest
  `TrustTransition` (by partition order). `REJECTED`/`QUARANTINED` ⇒ excluded from any
  trusted context / handoff slice.
- **lineage DAG** = nodes: claims; edges: `derived_from`/`supports`/`contradicts`.
  Key query: `descendants(claim_id)` = rollback blast radius (who relied on X).
- **agent state (resume)** = fold of `raw` for `agent_session_id` from the last
  `Snapshot.offset` forward → the messages array (topic 06). Must be a **pure function
  of the log** (canonical serialization; no ts/uuid/pod-id in the cached prefix).

## 4. Verifier contract (separate process — D1, topic 02)
- Consumes `raw`; selects claims to verify by **policy** (D1 Balanced: hub-role, at
  handoff boundaries, high-risk). Others stay `UNVERIFIED`.
- **Factuality** check: against `evidence_ref` / tools / world.
- **Faithfulness** check: entailment against the evidence bundle the producer was given
  (unsupported/contradictory ⇒ `REJECTED`).
- Emits `TrustTransition` (its own event; `verdict_derived_from` = claim + evidence).
- Circuit breaker: cap retries `K`; persistent-unresolved ⇒ `QUARANTINED` (high-risk
  tagged, excluded from trusted context).
- The verifier is itself an LLM step → its verdicts are auditable events (P0 "back up").

## 5. Handoff protocol (D2 + D3 boundary gate)
1. A emits `Handoff{seed_claim_ids, task_spec}`.
2. Runtime computes the **confirmed provenance subgraph**: seeds + supporting ancestors
   (`derived_from`, depth-bounded), confirmed-only. **Boundary gate (D3):** trigger
   on-demand verification of any unverified slice claims; await their trust-transitions.
3. Produce **Tier-0 synthesis**: a `Claim` (`derived_from` = the slice) — a verified LLM
   briefing (entail-checked against the slice; no new/distorted assertions). Logged as an
   event (for provenance + B's own resume — D2).
4. Spawn B (`Control{SPAWN}`). B's prompt = [B-role system] + [Tier-0 synthesis] +
   [task]. Tiers 1/2 (claims, evidence) available via **lazy expansion** on demand.
5. If a slice claim later flips `REJECTED`, emit `Control{REWIND, target=B-branch}` and
   re-derive.

## 6. Runtime loop (ties it together — topic 07 §7)
consume(raw@lineage_root) → fold state → assemble prompt (cache_control bps) →
LLM (opus-4-8, ttl:"1h") → emit stream_deltas → exec tools → emit tool_results →
emit self-declared claims (unverified) → on handoff run §5 → snapshot every N → commit.

## 7. Cross-cutting (garden's unsolved ops problems — design targets)
Exec/tool sandboxing; rate limiting (runaway agents); cost control (per-agent token
budgets — Anthropic `task_budget`); crash/failure recovery (resume covers the happy
path; need detection); multi-agent test harness (simulation). Tracked as their own
work area.
