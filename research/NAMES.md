---
doc_type: naming
topic: Component names and the rationale behind them
project: Greenwood
tags: [naming, greenwood, annals, rootlines, grieve, coldframe]
created: 2026-06-30
updated: 2026-06-30
status: active
summary: The five components of the system and why each is named as it is. Theme is arboreal/forest — chosen because the founding design insight was "preserve the tree of agentic interactions so faulty branches can be monitored and pruned." Names were checked free on crates.io / npm / PyPI.
---

# Names

The system is a **forest**. That framing isn't decoration — it comes from the founding
design insight: the bus must *preserve the tree/genealogy structure of agentic
interactions so faulty branches can be monitored and pruned* before errors cascade. Once
you're thinking in branches, roots, pruning, and cultivation, an arboreal naming set
falls out naturally and each name carries real meaning about the component's job.

All five were checked free on crates.io, npm, and PyPI at selection time (2026-06-30).

| Component | Name | What it names | Why the name |
|---|---|---|---|
| **The system** (event bus + agent runtime) | **Greenwood** | the whole living network agents run in — transport + runtime | a forest in leaf; the living wood the whole ecosystem inhabits. It's the top-level project name and the repo name. |
| **The immutable event log / spine** | **Annals** | the append-only, replayable record of everything that happens | annals are a chronological record of events year by year; the durable history you replay to reconstruct any state. |
| **The lineage / genealogy DAG** | **Rootlines** | the branching provenance graph — which claim derives from / supports / contradicts which | roots tracing ancestry; the underground structure that shows where everything came from. |
| **The verifier / governance layer** | **Grieve** | the process that screens claims, establishes trust, prunes faulty branches, isolates | a *grieve* is the Scots term for an estate overseer — the steward who walks the land, inspects it, and manages what grows and what gets cut back. |
| **The eval / refinement system** | **Coldframe** | feedback → replay evals → reproduce → tweak-harness → iterate → regression suite | a cold frame is the box where seedlings are *hardened off* — toughened against real conditions before being transplanted out. It's where behavior is proven before it ships. |

**How they fit together (one sentence):** agents live and work in **Greenwood**, every
action they take is recorded in **Annals**, their reasoning is connected by **Rootlines**,
**Grieve** decides what's trustworthy and prunes what isn't, and **Coldframe** turns
real-world failures into a standing test suite that keeps the whole wood healthy.

*This is a living map — add new components (and the reasoning behind their names) here.*
