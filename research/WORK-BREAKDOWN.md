---
doc_type: work-breakdown
topic: Greenwood — work breakdown (projects → milestones → tickets), source for Linear
project: Greenwood
tags: [work-breakdown, roadmap, linear, tickets, milestones, projects, mvp]
sources: [decisions, topics-05-08]
created: 2026-06-30
updated: 2026-06-30
status: draft — the SOURCE for Linear ticket creation (do NOT create in Linear until Terra authorizes)
summary: The Greenwood design decomposed into buildable units. 10 projects, 6 sequencing milestones, ~40 implementation-ready tickets. Each ticket has Why/Scope/Acceptance/Deps/Refs so it can become a Linear issue verbatim. Critical path: single-agent MVP → claims+governance → multi-agent handoff → eval + adapters → hardening.
---

# Greenwood — Work Breakdown

**Purpose:** this file is the SOURCE OF TRUTH for generating Linear tickets. Each entry
below is written to drop into a Linear issue (title + body). Design rationale lives in
`topics/` and `decisions.md` (referenced per ticket). **Do not create anything in Linear
until Terra explicitly authorizes it.**

**Linear mapping (proposed):** each `P#` → a Linear **project**; each `T#` → an
**issue**; the `M#` milestones cut across projects → Linear **milestones** (or cycles).
Terra confirms the exact structure before creation.

## Milestones (the critical path)
- **M1 — Single-agent MVP.** Event spine + runtime + resume for ONE agent; no
  claims/governance/handoff. Proves: event-sourced loop, crash-resume, cache-hit on
  resume. (P1, P2, P3)
- **M2 — Claims + governance.** Agent self-declares claims; verifier confirms; confirmed
  + lineage projections. Proves: event-sourced trust. (P4, P5)
- **M3 — Multi-agent handoff.** Genealogy handoff, boundary gate, rewind. Proves the
  differentiator: traceable, error-gated handoff. (P6, P7)
- **M4 — Agent eval & refinement.** Feedback events → auto-built replay evals →
  reproduce → tweak-harness → iterate → regression suite. Depends on M1 (replay) + M2
  (governance/lineage); can start once those land, before/alongside M3. (P9)
- **M5 — Harness adapters (Graft).** Protocol + SDKs + conformance + reference grafts
  (Claude Code, Codex, pi.dev, Hermes). Depends on M1 (envelope, runtime, resume). (P10)
- **M6 — Hardening.** Sandboxing, rate-limit, cost, observability, multi-agent test
  harness. (P8, plus hardening tickets across P1–P7)

Foundational principle behind all of it: **P0 — event sourcing end-to-end** (see
`decisions.md`).

---

## P1 — Event spine (Kafka substrate)  · M1
Refs: `topics/05-kafka-architecture-sketch.md`, `topics/08-concrete-spec.md` §1–2, D-log P0.

- **T1.1 — Event envelope schema (Protobuf + Schema Registry).**
  Why: every component speaks this; versioned to avoid the garden's no-version regret.
  Scope: define `Event` + payload variants (§1 of the spec); register in Schema Registry;
  codegen for the runtime language; content-hash `event_id` helper.
  Acceptance: round-trip encode/decode; backward-compat check passes; `event_id` is a
  pure function of (payload + causal position) per the canonical-hash spec (§1: pinned
  field order, ts/producer/epoch excluded, schema_version-scoped) with **cross-language
  hash test vectors**; `SerializationEpoch` control event defined. Deps: —.
- **T1.2 — Topic + partition layout.**
  Why: ordering is load-bearing (D-ref: Chattermax lost it). Scope: create the 6 topics
  (§2) with correct keys/retention/compaction; **key `raw` by `lineage_root`**; tiered
  storage (KIP-405) for `raw`/`control`. Acceptance: events for one `lineage_root` are
  totally ordered on one partition; compacted topics keep latest-per-key. Deps: T1.1.
- **T1.3 — Producer/consumer client library.**
  Why: one blessed way to read/write the spine. Scope: thin lib over rdkafka —
  idempotent producer (`acks=all`, `idempotence=true`), keyed publish, consumer with
  offset commit, replay-from-offset. Acceptance: publish→consume→replay integration test;
  duplicate publish is idempotent. Deps: T1.1, T1.2.
- **T1.4 — Content-hash idempotency + replay guarantees.**
  Why: resume/replay must never double-emit (topic 06/07). Scope: `event_id` = content
  hash incl. causal position; consumer dedup on replay; last-committed-offset tracking
  per `agent_session_id`. Acceptance: replaying a range emits no duplicates downstream.
  Deps: T1.1, T1.3.

## P2 — Agent runtime core  · M1
Refs: `topics/07` §1/§7, `topics/08` §6, `claude-api` skill (prompt caching, opus-4-8).

- **T2.1 — Runtime loop skeleton (stateless-recoverable).**
  Why: the component itself. Scope: consume→fold→assemble→LLM→emit loop (§6); holds no
  durable state not derivable from Kafka. Acceptance: a trivial agent completes a
  task end-to-end, all actions emitted as events. Deps: T1.3.
- **T2.2 — Anthropic LLM integration + deterministic prompt assembly.**
  Why: the call, with cache-correct prompt construction. Scope: `claude-opus-4-8`
  (adaptive thinking); assemble prompt in cache-friendly layers with `cache_control`
  breakpoints; **canonical serialization, no ts/uuid/pod-id in the cached prefix**;
  `ttl:"1h"`. Acceptance: two identical runs on different hosts show
  `cache_read_input_tokens > 0` on the second. Deps: T2.1.
- **T2.3 — Tool execution + result events.**
  Why: agents act. Scope: execute tool calls, emit `ToolResult` events (evidence, not
  claims). Acceptance: tool round-trip captured as events; replay reproduces it. Deps: T2.1.
- **T2.4 — Output streaming into Kafka (`StreamDelta`).**
  Why: salvage partial generations on crash (topic 06). Scope: stream LLM deltas as
  `StreamDelta` (stream_id/seq/complete). Acceptance: a killed mid-generation call leaves
  a replayable partial. Deps: T2.1, T1.3.

## P3 — Resume  · M1
Refs: `topics/06`, `topics/07` §4, D-log P0.

- **T3.1 — Snapshotting.** Scope: emit `Snapshot` every N turns (serialized state + offset
  + breakpoints); inline-or-object-store `state_ref`. Acceptance: snapshot present and
  points at a valid offset. Deps: T2.1.
- **T3.2 — Replay + deterministic state reconstruction.** Scope: load latest snapshot →
  replay `raw` after its offset → rebuild messages array **byte-identically**. Acceptance:
  reconstructed prompt is byte-equal to the pre-crash prompt (golden test). Deps: T3.1, T2.2.
- **T3.3 — Agent acquisition on reschedule + producer fencing.** Scope: consumer-group
  rebalance (small fleets) OR claim-on-`control` queue with expiry (scale); acquisition
  emits `Control{ACQUIRE}` bumping the `session_epoch`; folds reject stale-epoch events
  (topic 08 §2a). Acceptance: kill a pod mid-task → a new pod resumes and completes;
  **zombie test**: a paused-not-dead pod resumes producing after reassignment and its
  events are rejected by every fold. Deps: T3.2, T1.4.
- **T3.4 — Resume cache-continuity test.** Scope: end-to-end test that a resumed agent on
  a different host hits the warm cache (within the cadence-policy TTL — topic 06).
  Acceptance: `cache_read_input_tokens > 0` post-resume; a deliberately-introduced
  nondeterminism makes it 0 (guard). Deps: T3.2.

## P4 — Claims & lineage  · M2
Refs: `topics/03`, `topics/01`, `topics/08` §1/§3, D-log D1.

- **T4.1 — Agent self-declares claims (structured output).**
  Why: D1 — agent proposes claims + provenance. Scope: agent emits `Claim` events
  (unverified) with `derived_from`/tags as structured output in its turn; captures
  implicit reliance. Acceptance: a turn produces atomic claims with provenance edges;
  deterministic within the turn. Deps: T2.1, T1.1.
- **T4.2 — Lineage DAG projection.** Scope: projector consumer folds claims into
  `agent-bus.lineage` (adjacency); `descendants(claim_id)` query. Acceptance: DAG
  rebuildable from replay; descendants query returns the rollback blast radius. Deps: T4.1.
- **T4.3 — Claim tagging + deterministic grouping.** Scope: tags for topic/module;
  deterministic grouping (by tag/shared-ancestor) for handoff rendering — NO LLM
  summarization here. Acceptance: grouping is a pure function of claims+edges. Deps: T4.1.

## P5 — Governance / verifier  · M2
Refs: `topics/02`, `topics/08` §4, D-log D1/D3/P0, `spark-to-fire` review.

- **T5.1 — Verifier process (separate consumer).**
  Why: D1 — trust established independently, beside the bus. Scope: consumes `raw`,
  policy-selects claims, runs factuality + faithfulness (entailment) checks, emits
  `TrustTransition` events. Acceptance: a false claim → `REJECTED`; a supported one →
  `CONFIRMED`; verdict carries `verdict_derived_from`. Deps: T4.1.
- **T5.2 — `confirmed` projection.** Scope: projector folds `TrustTransition` → compacted
  `agent-bus.confirmed` (latest state per claim); rejected/quarantined excluded from
  trusted context. Acceptance: projection = fold of the log; rebuildable by replay. Deps: T5.1.
- **T5.3 — Verification policy (selective).** Scope: Balanced policy — verify hub-role,
  at-boundary, high-risk claims; others stay unverified until needed; risk scoring.
  Acceptance: only policy-selected claims are verified; on-demand verification triggers
  at boundaries. Deps: T5.1.
- **T5.4 — Circuit breaker.** Scope: cap retries K; persistent-unresolved → `QUARANTINED`
  (excluded from trusted context — never forwarded-with-tag, per D3 refinement).
  Acceptance: a claim that can't be resolved is quarantined, not retried forever. Deps: T5.1.
- **T5.5 — Provenance audit (sampled re-decomposition).** Why: D1's edges are
  self-declared and game-able; the audit restores independence. Scope: Grieve
  periodically decomposes a sampled agent turn bus-side, diffs claims/edges against the
  agent's self-declared set, emits a divergence event that adjusts that agent's
  verification rate. Acceptance: a deliberately under-declaring test agent is detected
  within N sampled turns; its verification rate rises. Deps: T5.1, T4.1.
- **T5.6 — Effect gate.** Why: trust must govern actions, not just handoffs (topic 08
  §5a). Scope: tool registry declares `effect_class` per tool; runtime blocks
  IRREVERSIBLE tool calls whose premise claims aren't confirmed (or escalates to human);
  REVERSIBLE requires a registered compensation handler. Acceptance: an unverified-premise
  irreversible call is held pending verification; a reversible one runs and its
  compensation is invoked on branch rewind. Deps: T5.1, T2.3.

## P6 — Multi-agent handoff (the differentiator)  · M3
Refs: `topics/07` §5, `topics/08` §5, D-log D2/D3.

- **T6.1 — Handoff event + confirmed subgraph selection.** Scope: `Handoff` event; runtime
  computes confirmed provenance subgraph (seeds + depth-bounded ancestors). Acceptance:
  subgraph is confirmed-only, deterministic, depth-bounded. Deps: T4.2, T5.2.
- **T6.2 — Boundary gate (D3).** Scope: on-demand verify unverified slice claims; await
  their trust-transitions before spawn. Acceptance: no unconfirmed claim crosses the
  boundary; local await, not global block. Deps: T6.1, T5.3.
- **T6.3 — Tier-0 verified synthesis.** Scope: LLM synthesis of the slice as a `Claim`
  (`derived_from` the slice), entail-checked against sources; logged as an event.
  Acceptance: synthesis asserts only what the slice entails (injected hallucination →
  rejected); logged + provenance-linked. Deps: T6.1, T5.1.
- **T6.4 — Lazy expansion (Tiers 1/2).** Scope: B expands synthesis → claims → evidence on
  demand. Acceptance: B can drill from briefing to underlying confirmed claims to evidence.
  Deps: T6.3, T4.2.
- **T6.5 — Rewind on rejection.** Scope: when a handed claim flips `REJECTED`, find
  `descendants` and emit `Control{REWIND}` on affected branches; re-derive. Acceptance:
  a post-hoc rejection surgically rewinds only affected branches, via compensating events
  (no deletion). Deps: T6.1, T4.2, T7.1.

## P7 — Lifecycle & control  · M3
Refs: `topics/04`, `topics/08` §1(Control), D-log P0.

- **T7.1 — Control vocabulary (spawn/fork/rewind/abandon/freeze/resume).** Scope: `Control`
  events on `agent-bus.control`; agent-trajectory tree projection (Gardener-style
  Fork/Rewind/Promote/Abandon). Acceptance: lifecycle transitions are events; tree
  rebuildable by replay. Deps: T1.3.
- **T7.2 — Swarm / fan-out.** Scope: one parent spawns N siblings sharing a handoff slice;
  concurrency control. Acceptance: N implementers run in parallel off one slice. Deps:
  T6.3, T7.1.
- **T7.3 — Failure recovery (crash detection).** Scope: detect crashed/timed-out agents
  (the garden's open Q#7); trigger resume or abandon. Acceptance: a stuck agent is
  detected and recovered/abandoned, not left dangling. Deps: T3.3, T7.1.

## P8 — Ops & hardening  · M6
Refs: `topics/08` §7 (cross-cutting ops problems).

- **T8.1 — Tool/exec sandboxing.** (Docker/nspawn/seccomp — the garden left this open.)
- **T8.2 — Rate limiting** (per-agent, prevent runaway spam).
- **T8.3 — Cost control** (per-agent token budgets via Anthropic `task_budget`).
- **T8.4 — Observability** (trace an outcome through the lineage DAG; dashboards over the
  event stream — a read-only projection, topic 02).
- **T8.5 — Multi-agent test/simulation harness** (the garden's open Q — test swarms
  deterministically by replaying event logs).

## P9 — Agent eval & refinement  · M4
Refs: `topics/09-agent-eval.md`, `topics/06` (replay), D-log P0/D4. Realizes the garden's
Hotspots/Chizu refinement loop on the bus.

- **T9.1 — Feedback events + ingestion.**
  Why: register human/automated feedback in the log stream as the eval seed. Scope:
  `Feedback` event + `agent-bus.feedback` topic (keyed `lineage_root`); API/UI hook to
  attach thumbs + correction to a target event; wire verifier `REJECTED` + implicit
  signals (tests/PR/task-failed) as feedback sources; provenance `derived_from` target.
  Acceptance: a "thumbs down; should've done X not Y" lands as a provenance-linked event.
  Deps: T1.1, T4.2.
- **T9.2 — Eval-builder (feedback → EvalCase).**
  Why: turn a complaint into a replayable test. Scope: consumer that freezes the context
  (event range Snapshot→target offset) and derives an assertion (structured or
  rubric/LLM-judge) → emits an `EvalCase` event. Acceptance: a thumbs-down yields a
  replayable EvalCase carrying frozen-context-ref + assertion + provenance. Deps: T9.1, T3.2.
- **T9.3 — Hermetic eval-runner.**
  Why: reproduce deterministically. Scope: replay/eval-mode agent pointed at a frozen
  context, feeding recorded `ToolResult`s as fixtures (D4 hermetic); assert against
  rubric/judge; N-run pass-rate. Acceptance: reproduces the original failure (must fail
  first); emits a pass-rate metric. Deps: T9.2, T3.2, T2.2.
- **T9.4 — Harness versioning + refine loop.**
  Why: tweak-and-iterate-until-pass. Scope: versioned harness bundle (prompt/tools/
  policy/model/effort/skills); run EvalCase against a harness version; iterate loop
  (human or meta-agent, tweaks as events). Acceptance: a fixing tweak flips the eval
  red→green; harness versions + runs recorded as events. Deps: T9.3.
- **T9.5 — Regression suite + behavior CI.**
  Why: don't regress fixed cases. Scope: standing suite; re-run selected subset on each
  harness change (selection by tag/subgraph, like D2); pass-rate dashboard (projection).
  Acceptance: a harness change re-runs affected evals and surfaces regressions. Deps: T9.4.
- **T9.6 — Failure clustering via lineage.**
  Why: target root cause, cover a class. Scope: cluster feedback/failures by
  `descendants`/tags; one eval covers a cluster; blast-radius query drives targeting.
  Acceptance: related failures group; a root-cause eval validates against the cluster.
  Deps: T9.2, T4.2.

## P10 — Graft (harness adapters)  · M5
Refs: `topics/10-graft-harness-adapters.md`, `topics/08-concrete-spec.md`, D-log D5.

- **T10.1 — Graft protocol spec.**
  Why: the stable, language-neutral contract every harness maps onto. Scope: Protobuf
  event envelope (shared with T1.1) + a gRPC lifecycle service
  (`init / step / apply_tool_result / snapshot / resume / stop`) + a capability
  descriptor. Acceptance: service compiles; a trivial echo graft drives a session
  end-to-end. Deps: T1.1.
- **T10.2 — Graft SDK (first language).**
  Why: authors write only translation + lifecycle, not Kafka plumbing. Scope: base
  trait/class + Kafka wiring (produce/consume, offsets, snapshot storage) + determinism
  helpers (canonical serialization, breakpoint placement, "no wall-clock in prefix"
  lint). Acceptance: a reference graft built on it passes conformance. Deps: T10.1, T1.3.
- **T10.3 — Conformance suite.**
  Why: pass it → resume/handoff/evals work with no per-harness special-casing. Scope:
  event-mapping round-trip; correct-state resume; byte-identical resume (cache-continuity
  tier); handoff injection; capability honesty. Acceptance: reports per-tier pass/fail for
  a graft. Deps: T10.2, T3.2.
- **T10.4 — Reference graft: Claude Code.** Deps: T10.2, T10.3.
- **T10.5 — Reference graft: Codex.** Deps: T10.2, T10.3.
- **T10.6 — Reference grafts: pi.dev + Hermes Agent.** Deps: T10.2, T10.3.
- **T10.7 — Graft SDKs: TypeScript / Python / Go.**
  Why: write adapters in the harness's own language. Scope: port the SDK surface +
  conformance harness to each. Deps: T10.1, T10.3.
- **T10.8 — Capability negotiation + harness-version tracking.**
  Why: adapt runtime behavior per harness; handle resume across harness upgrades. Scope:
  capability descriptor consumed by the runtime; harness version stamped on events; a
  version bump drops resume to correctness-only. Deps: T10.1.

---

## Open questions to resolve before/while ticketing
- Runtime **language/stack** (topic 05): Kafka Streams is JVM-first; Rust needs rdkafka +
  hand-rolled stream logic. Blocks P1/P2 shape. **Needs Terra.**
- Lineage store: Kafka-compacted-topic only vs. + external graph DB (Memgraph/Neo4j) for
  rich `supports`/`contradicts` queries (topic 01). Affects T4.2/T8.4.
- Verifier's own trust model (who verifies the verifier? — P0 says its verdicts are
  events, but escalation policy is open).
