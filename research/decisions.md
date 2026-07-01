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
repaired; unresolved follow circuit-breaker policy (forward high-risk-tagged or exclude).
If B was spawned optimistically and a slice-claim later flips to `rejected`, emit a
rewind event on B's branch. Local synchronization, not global inline-blocking.

**Why not inline:** inline blocking fights Kafka's async grain and puts verification
latency in every critical path; the safety inline would buy is recovered by the boundary
gate (unconfirmed claims simply never enter a *trusted* context) without stalling the
whole system.

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

**Residual risk (accepted, monitor):** an agent could lie about provenance to hide an
error's origin. Bounded because (a) its real context is in the log and auditable, and
(b) governance verifies *content* independently, so a mis-attributed-but-false claim is
still rejected on its merits.

**Related:** [[topics/03-atomic-claim-envelope]], [[topics/07-runtime-design-resume-genealogy-handoff]]

**Still open (feeds later decisions):** D2 handoff granularity (claims vs. clustered
findings); D3 governance timing (inline vs. async quarantine-until-confirmed).
