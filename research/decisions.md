---
doc_type: decisions-log
topic: Greenwood design decisions (ADR-style running log)
project: Greenwood
tags: [decisions, adr, claims, decomposition, governance, handoff]
sources: [spark-to-fire, claude-api-skill, engineering]
created: 2026-06-30
updated: 2026-06-30
status: active — proposed decisions pending Terra's final confirmation
summary: Running log of resolved/proposed design decisions for Greenwood, most recent first. Each entry: the decision, the options, and the reasoning.
---

# Greenwood — Decisions Log

*Ordering: P0 (the foundational principle) is pinned first; the D-entries below it run
newest-first, append-at-the-top — read bottom-up (D1→D6) for chronological order.*

## P0 — FOUNDATIONAL PRINCIPLE: event sourcing end-to-end  (Terra, 2026-06-30)

**Everything is event sourcing, top to bottom and back up again.** Every input, every
derived state, every control action, every correction is an **event** on the log; the
only "state" anywhere is a **fold/projection** of the log. There is NO out-of-band
mutable state — not trust, not lifecycle, not snapshots. Concretely:
- Agent actions, claims, tool results, handoffs → events.
- **Trust is event-sourced**: a claim's `trust_state` is the fold of trust-transition
  events (`proposed`→`verified`→possibly `rejected`), NOT a mutable field. Trust changes
  over time with full history → rollback = emit a `rejected`/compensating event + re-fold,
  never mutate/delete.
- The `confirmed` topic is a **projection** (compacted view), not the source of truth.
- The verifier's verdict is an event `derived_from` the claim + evidence → the verifier
  is itself auditable/governable ("back up again").
- Lifecycle (spawn/fork/rewind/abandon) and snapshots are also events.
This is what makes resume (state = fold), handoff (slice = fold as-of offset), and
rollback (compensating events + re-fold) all the *same* mechanism.

**Scope honesty (added 2026-07-02, red-team):** the shared mechanism governs *derived
state* — belief. It does **not** rewind the world (a pushed commit, a sent email), and
"re-derive" of an LLM branch is stochastic *regeneration* at inference cost, not a
deterministic re-fold. Resume replays a recorded past; rollback repairs belief,
quarantines descendants, runs registered compensation for reversible external effects,
and surfaces irreversible ones for humans. Actions are therefore trust-gated too — see
the **effect gate** (topic 08 §5a): irreversible tool effects require confirmed premises.
This is recorded as its own design area (effect classes, compensation handlers).

---

## D6 — Backing store: keep the Kafka API, run it object-store-native  (proposed 2026-07-01)

**Decision (proposed):** Target the **Kafka API/abstraction** (append log, keyed ordering,
replay, compaction, tiered retention) — but do **not** run on vanilla open-source Kafka.
Greenwood's workload is write-heavy with rare/cold reads, and vanilla Kafka's bill is
dominated by exactly the wrong things for that shape: hot storage × 3× replication, and
inter-AZ replication traffic (~$0.04/GB ingested — 2 follower streams, both directions
billed — often >50% of a Kafka bill). Read/egress — Kafka's expensive axis — is ~$0 here.
Vanilla also can't tier compacted topics (KIP-405 excludes them), so the compacted
projections would be pinned to replicated local disk.

Run it on an **object-store-native, Kafka-compatible engine**: **AutoMQ** (Kafka fork —
full compaction via Kafka's real LogCleaner, works on S3-backed storage so the KIP-405
restriction doesn't apply; sub-10ms P99 single-AZ EBS WAL (~20ms multi-AZ); Apache 2.0
since Apr 2025; drop-in) or **WarpStream** (managed, drop-in — but mind compaction
ceilings: 3M distinct keys / 128 GiB per job, immutable `cleanup.policy`; vendor note:
Confluent-owned, and IBM's acquisition of Confluent closed 2026-03-17). Both drop
triple-replicated NVMe for S3 and remove cross-AZ replication → storage/network ~5–10×
cheaper, **design + Graft adapters unchanged**. **Raw S3-as-log** is cheapest at scale
but a full rewrite — fallback only. Rejected: Kinesis / Pub/Sub (no compaction; 365d/31d
retention ceilings); NATS JetStream (no cold/object-store tier; per-subject last-N
retention is not Kafka-style compaction); Pulsar (topic compaction fails on S3 tiered
storage — apache/pulsar#8282, unfixed).

**Also adopt (engine-independent):** payloads-by-reference (large tool outputs → S3 blob +
hash in the event; ingest-vs-store gap ≈3,300×, derived from W&B Weave's $0.10/MB ingest
vs $0.03/GB·mo storage), snapshot-to-bound-hot-retention, and cold-tier to S3-IA/Glacier
for the multi-year archive.

**Cost shape (from the 500-agent-year model, topic 11):** at 500 agents Greenwood is a
*small* throughput workload (sub-10 MB/s) — storage- and compute-floor-bound, not
throughput-bound. State capture ≈ **low tens of $k/yr**. The **eval (Coldframe) bucket is
LLM-inference-dominated and dwarfs storage** at any real regression cadence.

**Open:** measure produce latency on the S3 path. **Resolved (2026-07-02, by analysis):**
compaction-vs-cardinality — the claim keyspace is *monotone* (each `claim_id` written
~2–3× then never again), so latest-per-key compaction converges to the full claim set at
fleet scale (~112 TB/yr at 500k agents; WarpStream's 3M-keys/partition budget exhausted in
hours). Compaction stays for bounded keyspaces (snapshots, agent lifecycle); claim state
needs **TTL-after-terminal-state** (archive + drop from the compacted view) or a DB-backed
projection — pick one before M2 (see topics 05/08).

**Related:** [[topics/11-scalability-and-cost]], [[topics/05-kafka-architecture-sketch]],
[[topics/10-graft-harness-adapters]].

---

## D5 — Harness adapters: the Graft protocol (harness-agnostic)  (proposed 2026-07-01)

**Decision (proposed):** Greenwood is **harness-agnostic**. Its only external contract is
the **event envelope + a lifecycle protocol**; any harness (Claude Code, Codex, pi.dev,
Hermes, …) plugs in through a **Graft** adapter — the only per-harness code. Ships as: a
protocol spec (Protobuf envelope + gRPC lifecycle), per-language SDKs (Rust/TS/Python/Go),
a conformance suite, and reference grafts.

**Contract a Graft implements:** translate harness actions → events (claims optional —
Greenwood can decompose from raw output, so a harness needn't be claim-aware); lifecycle
`init / step / snapshot / resume / stop`; accept a handoff payload as starting context.

**Two-tier conformance:** **correctness** (resume rebuilds correct state — required) vs.
**cache-continuity** (byte-identical prompt so the content-addressed cache hits —
best-effort). Honest split: not every third-party harness can hit byte-identity; those
resume correctly and just pay a prefill.

**Deployment:** sidecar (gRPC) by default; in-process for same-language embedding. Tool
execution can sit on either side — the recorded events are identical (Coldframe hermetic
replay depends on this).

**Name:** Graft — a scion (the harness) grafted onto a rootstock (Greenwood). Registry
availability not yet checked.

**Related:** [[topics/10-graft-harness-adapters]], [[topics/06-agent-resume-and-caching]],
[[topics/08-concrete-spec]].

---

## D4 — Eval reproduction: hermetic replay, evals-as-events, statistical pass-rate  (proposed 2026-06-30)

**Decision (proposed):** Agent evals are **seeded by feedback events** and built by
**replaying the frozen context** (event range) from P0's log — reusing the resume/replay
machinery (P3), not a separate eval infra. Three sub-choices:
- **Hermetic replay by default:** feed the **recorded `ToolResult` events as fixtures**
  (they're already in the log as evidence) rather than re-executing tools → deterministic
  inputs, only the model varies. (Non-hermetic live-tools mode is an option for
  integration evals, but flaky.)
- **Evals are events (P0):** `Feedback`, `EvalCase`, harness versions, and eval runs are
  all events → the eval suite + refine loop are versioned, auditable, replayable.
- **Statistical, not exact:** an eval asserts "satisfies rubric/assertion," run **N times
  with pass-rate ≥ threshold** (LLMs are stochastic even on identical input).

**Why:** the event-sourced spine makes a failure *already* a replayable frozen context;
feedback pins the point; the eval loop = replay-through-harness → must-fail (reproduce) →
tweak versioned harness → test → iterate → regression suite. Realizes the garden's
Hotspots/Chizu validate→refine loop on the bus. Lineage DAG lets evals target root cause
(`descendants` blast radius) and cluster similar failures.

**Related:** [[topics/09-agent-eval]], [[topics/06-agent-resume-and-caching]]; feedback
sources include the verifier's own `REJECTED` transitions (D1).

---

## D3 — Governance timing: inline vs. async  (proposed 2026-06-30)

**Decision (proposed):** **Async.** Claim confirmation is a separate **event** (a
trust-transition), per P0. The pipeline never blocks globally — claims flow immediately;
an agent may use its *own* unverified claims as working memory. Consumers needing trusted
context read the **`confirmed` projection**, never `raw`.

**Boundary gate (reconciles async with "nothing unconfirmed crosses a handoff"):** at a
**trust boundary** (handoff / entering another agent's confirmed context) there is a
*local* await — the handoff triggers on-demand verification of its slice and B spawns
once the slice's verdict-events resolve. Confirmed claims cross; rejected are dropped/
repaired; unresolved are **excluded** (refined 2026-07-02: never "forward with a
high-risk tag" — an LLM reading the briefing won't reliably discount a tag).
**Optimistic spawn** (B starts before the slice fully resolves) is confined to work
whose tool effects are `PURE`/`REVERSIBLE` (see the effect gate, topic 08 §5a); if a
slice-claim later flips to `rejected`, emit a rewind event on B's branch — belief repair
+ compensation, per P0's scope-honesty note. Local synchronization, not global
inline-blocking. "Read `confirmed`, never `raw`" is **ACL-enforced**, not conventional
(topic 08 §2b).

**Why not inline:** inline blocking fights Kafka's async grain and puts verification
latency in every critical path; the safety inline would buy is recovered by the boundary
gate (unconfirmed claims simply never enter a *trusted* context) without stalling the
whole system. **Honest ledger note:** handoffs *are* a critical path in a multi-agent
system, and the gate puts LLM calls (slice verification + synthesis + entailment) in
each one — the latency moved from every-message to per-handoff; it didn't vanish.
Gate latency is a measured budget item (see topic 11's governance-inference bucket).
The verifier itself is monitored for drift (its verdicts are events; a periodic audit
re-verifies a sample against ground truth — the circuit breaker covers claims, not a
drifting verifier).

**Related:** [[topics/02-bus-vs-governance-boundary]], [[topics/07-runtime-design-resume-genealogy-handoff]]

---

## D2 — Handoff granularity: what crosses an A→B handoff  (proposed 2026-06-30)

**Decision (proposed):** **Atomic claims stay the substrate** (never coarsened — trust,
provenance, rollback all live there). The handoff payload is **graduated / progressive
disclosure**, all tiers provenance-linked:
- **Tier 0 (first pass):** LLM **selection + synthesis** → a compact, coherent briefing
  for B, carrying `derived_from` edges to the confirmed source claims.
- **Tier 1 (on demand):** expand a point → the underlying **confirmed atomic claims**
  (the subgraph).
- **Tier 2 (on demand):** expand a claim → its **`evidence_ref`** (tool result / source).

**Guardrail:** the Tier-0 synthesis is **verified against its source claims**
(entailment: asserts only what the confirmed set entails — no hallucinated/distorted
claims) BEFORE it crosses to B. Lazy expansion complements this, it does NOT replace it
— B reasons over the briefing first and could act on a distortion before expanding.
*(Precision note, 2026-07-02: the entailment check is itself an LLM judgment — a
stochastic filter with a nonzero false-pass rate at exactly the boundary the system
protects. Its residual error rate is a monitored quantity; high-risk handoffs run two
independent checkers.)*

**The synthesis is not a special "bus summary" feature** — it's just another governed
agent output (a claim), produced by a process beside the bus, subject to D1's
decompose/verify/provenance rules, and **logged as an event**.

**Caching is NOT a design driver at handoff (Terra):** a fresh recipient B has a cold
cache, so the payload is a first write, not a read — we don't engineer its bytes for a
warm cache. This is what makes LLM synthesis viable here (removes the determinism-for-
caching constraint). BUT still **log the synthesis as an event** for (a) provenance and
(b) B's *own* later resume (replay reads the logged slice; never re-run the LLM on
resume, which would produce a different slice and blow B's accumulated cache).

**Options considered:** atomic-claims-only (traceable, floods B, salad); clustered
"findings" via deterministic tag/graph grouping (readable, lossless, but crude); LLM
summarization (readable, smart — but unguarded is an unscreened generator at the
governance boundary → rejected as default). Chosen = atomic substrate + verified LLM
synthesis as a selective, provenance-linked, expandable enhancement.

**Rejected:** unguarded LLM summarization handed to B as trusted prose (re-introduces
error propagation at the exact boundary the design exists to protect; the abstraction
leak). Also rejected: "an LLM in the bus" — the bus stays transport; synthesis is a
process beside it.

**Default vs. enhancement:** deterministic projection + lazy expansion is the cheap
floor; verified synthesis is the selective enhancement where B's context-quality payoff
justifies the extra governed LLM step.

**Related:** [[topics/03-atomic-claim-envelope]], [[topics/07-runtime-design-resume-genealogy-handoff]]

---

## D1 — Who decomposes claims, and when  (proposed 2026-06-30)

**Decision (proposed):** **Hybrid.** The producing agent **self-declares** its atomic
claims + provenance edges (`derived_from`/`supports`/`contradicts`) as structured output
inside its turn; those claims land as `trust_state = unverified`. An **independent
governance layer** assigns the trust state (`confirmed`/`rejected`) — the agent never
sets its own truth-value. Decompose **universally** (cheap, agent-side); **verify
selectively** (expensive, bus-side) on the paper's Balanced policy — hub-role claims, at
handoff boundaries, and high-risk claims; the rest sit `unverified` and are verified
on demand when they'd enter a trusted context / handoff slice.

**Three distinct parties (Terra, confirmed):** (1) the **agent** (in the runtime) makes
claims; (2) a **separate verifier process** — a consumer of the bus — establishes trust
and writes the `confirmed` topic; (3) the **bus** (Kafka) is pure transport + immutable
log and does neither. Verification is NOT bus logic; the verifier sits beside the bus as
a coordinator (transport-not-merge boundary, [[topics/02-bus-vs-governance-boundary]]).

**Options considered:**
- **A — agent self-declares (only):** cheapest, and the only party that knows its
  *implicit reliance*; but self-report can't be trusted for truth-value, and is
  inconsistent across agents (breaks determinism).
- **B — bus-side decomposer (only):** independent + canonical/consistent; but expensive
  (LLM pass per message) and *worse* at implicit reliance (sees only output, not
  reasoning). This is the paper's approach.
- **C — hybrid (chosen).**

**Reasoning:** A claim carries two separable things with opposite trust properties —
**provenance** (the agent is the authority; it reports its own reasoning) and
**truth-value** (the agent is the audited party; must be established independently). C
assigns each column to the right authority: agent proposes structure+provenance, bus
establishes trust. Inherits A's cheapness + reliance-knowledge and B's independence.
Preserves the determinism spine (self-declared claims are in-turn structured output →
replay identically; `confirmed` is a governance-emitted event → also replays).

**Divergence from the paper (justified):** the paper decomposes bus-side because it's a
plug-in over *unmodifiable* MAS frameworks. We build the runtime, so we can make the
agent emit its own claims — getting provenance from the source instead of
reverse-engineering it. A greenfield advantage.

**Residual risk (revised 2026-07-02, red-team):** an agent could lie about provenance to
hide an error's origin — and the original bound ("auditable log + independent content
checks") is insufficient on its own: content is only checked for the *selectively
verified* subset, and the selection policy (hub/centrality) is computed from the same
self-declared edges the audited party controls. Split a hub claim into ten, or point
edges at innocuous parents, and you stay under the verification radar; a rejected claim's
`descendants()` then under-cuts the rewind. **Mitigations (adopted):**
- **Sampled independent re-decomposition audits** — Grieve periodically decomposes raw
  output bus-side (the paper's mechanism, as a spot-check) and diffs against the agent's
  self-declared claims/edges; divergence is itself an event that lowers the agent's (or
  graft's) trust and raises its verification rate.
- **Conservative rewind fallback** — blast radius = `descendants(claim)` ∪ everything in
  the lineage after the rejected claim's offset (topic 08 §3).
- Centrality/risk-based verification selection must never be computed *solely* from
  self-declared edges (mix in offset-order, adoption counts from handoffs, and audit
  divergence history).

**Related:** [[topics/03-atomic-claim-envelope]], [[topics/07-runtime-design-resume-genealogy-handoff]]

**Still open (feeds later decisions):** D2 handoff granularity (claims vs. clustered
findings); D3 governance timing (inline vs. async quarantine-until-confirmed).
