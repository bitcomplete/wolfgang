---
doc_type: index
project: Greenwood
purpose: Design research and work breakdown for Greenwood — an event-sourced agent runtime + coordination bus
created: 2026-06-30
updated: 2026-06-30
retrieval: every note has YAML metadata in its top 10 lines (head -n10 <file>)
structure: topics/ = thematic notes; papers/ = literature reviews
status: active design
---

# Greenwood — Research

**Greenwood** is an event-sourced agent runtime and coordination bus, built on Kafka.
The design driver: an agent bus should be **more than a flat event stream** — it must
**preserve the tree/genealogy structure of agentic interactions** so faulty branches can
be **monitored and pruned** before errors cascade into "false consensus." That instinct
is grounded in the *Spark to Fire* paper (see `papers/`), whose lineage-graph governance
layer maps directly onto this design.

Component names and the reasoning behind them are in **`NAMES.md`**
(Greenwood · Annals · Rootlines · Grieve · Coldframe · Graft).

> **A note on prior-art references.** Several topic notes compare Greenwood against
> earlier internal Bit Complete design explorations — a family of prototypes we call
> "the software garden" (names like *Chattermax*, *Chalet*, *Chizu*, *Gardener* appear
> throughout). Those systems are referenced purely as prior art and lessons learned;
> Greenwood is a fresh design with no dependency on them, and no familiarity with them
> is needed to read these notes.

## How these notes are organized
- Every file carries YAML front-matter in its **first 10 lines**, so it can be
  retrieved/triaged with `head -n10 <file>`.
- `topics/` — one note per design theme.
- `papers/<slug>/` — a literature review per source paper.

## Reading order
1. `decisions.md` — the settled design decisions (P0 event-sourcing; D1 claims/verifier;
   D2 spawn briefs; D3 async; D4 evals; D5 Graft; D6 backing store; D7 bus-side
   decomposition; D8 messaging + per-message policy). **Start here.**
2. `NAMES.md` — the component names and what each one is.
3. `topics/07-*` — the unified runtime design (resume + the spawn-brief pattern; see D8).
4. `topics/08-concrete-spec.md` — the buildable layer (envelope, topics, folds, the
   Grieve policy cascade, messaging).
5. `WORK-BREAKDOWN.md` — projects → milestones → tickets.
Topics 01–06 and 09 are supporting design rationale.

## Key files
- `decisions.md` — ADR-style decisions log (P0, D1–D8)
- `NAMES.md` — component names + rationale
- `WORK-BREAKDOWN.md` — projects / milestones / tickets

## Topics
- `topics/01-provenance-vs-event-stream.md` — causal/lineage DAG vs. flat log
- `topics/02-bus-vs-governance-boundary.md` — transport-not-merge vs. intervention
- `topics/03-atomic-claim-envelope.md` — event envelope / message granularity
- `topics/04-branch-pruning-agent-lifecycle.md` — pruning claims vs. pruning agents
- `topics/05-kafka-architecture-sketch.md` — Greenwood on Kafka
- `topics/06-agent-resume-and-caching.md` — resume after pod reschedule, preserving LLM prompt-cache hits
- `topics/07-runtime-design-resume-genealogy-handoff.md` — the core unified design
- `topics/08-concrete-spec.md` — buildable layer: envelope, topics, fold rules, verifier, handoff protocol
- `topics/09-agent-eval.md` — feedback-seeded, replay-based evals (Coldframe)
- `topics/10-graft-harness-adapters.md` — Graft: pluggable adapters for any agent harness
- `topics/11-scalability-and-cost.md` — scalability profile, Kafka alternatives, and a 500-agent-year cost model

## Papers
- `papers/spark-to-fire/` — Xie et al. 2026, "From Spark to Fire: Modeling and
  Mitigating Error Cascades in LLM-Based Multi-Agent Collaboration"
  ([arXiv:2603.04474](https://arxiv.org/abs/2603.04474))
