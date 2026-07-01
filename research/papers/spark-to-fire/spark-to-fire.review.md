---
doc_type: literature-review
title: "From Spark to Fire: Modeling and Mitigating Error Cascades in LLM-Based Multi-Agent Collaboration"
authors: Xie, Zhu, Zhang, Zhu (T.), Ye, Qi, Chen, Zhou
venue: arXiv:2603.04474v2 (cs.MA), 11 May 2026; Faculty of Data Science, City Univ. of Macau + Minzu Univ.
url: https://arxiv.org/abs/2603.04474
project: Greenwood
tags: [error-cascade, false-consensus, lineage-graph, genealogy, provenance, rollback, multi-agent, governance, propagation-dynamics]
created: 2026-06-30
updated: 2026-06-30
status: reviewed-in-full (15 pp)
relevance: HIGH — formalizes the "preserve tree-structure, prune faulty branches" requirement for Greenwood
summary: A single atomic error propagates via context reuse into system-level false consensus; defense = a genealogy/lineage-graph governance middleware that decomposes messages into atomic claims, tri-state screens them against confirmed provenance, and BLOCKS+ROLLS BACK faulty branches. Detection alone is insufficient — rollback/isolation is essential.
---

# Literature Review — "From Spark to Fire"

## Thesis
In LLM Multi-Agent Systems (LLM-MAS), a minor inaccuracy ("atomic falsehood" `m`)
can **propagate through iterative context reuse** and **solidify into system-level
false consensus** — a stable lock-in of a wrong belief. Errors are hard to trace
because they undergo semantic shifts during transmission. The paper (a) models the
propagation as system dynamics, (b) identifies structural vulnerabilities, (c)
instantiates an attack, and (d) proposes a governance defense. ≥89% (up to 100%)
infection prevention.

## Model (§II)
- Message flow as **directed graph `G=(V,E)`**, adjacency `A`, in-neighbors `N(i)`.
- **Atomic falsehood** `m`: minimal declarative claim violating correctness.
- **Propagation as Adoption** (Def 2): agent *adopts* `m` as a **semantic commitment
  / functional premise** (direct entailment OR implicit reliance) — not mere
  surface repetition. Binary `Xᵢ(t)`; adoption prob `sᵢ(t)=E[Xᵢ(t)]∈[0,1]`.
- **False consensus** (Def 3): system coverage `S(t)=mean sᵢ(t)` exceeds threshold
  `τ` and stays (error lock-in, distinct from transient fluctuation).
- Two error classes:
  - **Factuality error** — contradicts external ground truth / oracle / verified
    tool output.
  - **Faithfulness error** — unsupported by / contradictory to the **evidence
    bundle `Eᵢ(t)`** the agent was given (task-unfaithful or context-inconsistent).
- Infection function via **Independent Cascade + mean-field (IBMF)**; product-form
  `f^prod`. One-step: `sᵢ(t+1)=(1−δ)sᵢ(t)+(1−sᵢ(t))fᵢ(·)`, where `δ` = effective
  decay/self-correction.
- **Spectral early-warning Risk Criterion**: linearize early stage →
  `R ≈ βρ(A)/δ`; amplification when **`βρ(A) > δ`** (`ρ(A)`=spectral radius). A
  diagnostic, not a calibrated predictor.

## Three endogenous vulnerabilities (§IV)
1. **Cascade Amplification** — weak early self-correction + multi-neighbor exposure
   compounds (non-linear `1−∏(1−βaⱼ sⱼ)` term); supercritical regime saturates.
2. **Topological Fragility** — **hub** nodes are catastrophic injection points. Most
   dangerous seed = dominant eigenvector node `v*=argmax[u₁]_v`. Measured Impact
   Factor **6.29× (CrewAI) – 10.31× (LangGraph)** hub-vs-leaf. *Where* matters.
3. **Consensus Inertia** — correction cost rises as workflow progresses; intermediate
   artifacts crystallize errors into constraints ("contextual debt", "accumulated
   polluted rounds"). Late interventions fight accumulated history.

## Exogenous attack (§V)
Adversary injects minimal seeds at high-impact (spectral) positions, packaged for
credibility: **Compliance** ("per company policy", "verified by admin") and
**Security_FUD** ("emergency patch for CVE-2024-0001"). App-layer only (no host
exec, no training poison). Intent-hiding packaging pushes ASR to 76.7–100%.

## Defense — Genealogy-Based Governance Layer (§VI) ★ core for Greenwood
Plug-in **middleware on the message path**; keeps topology `A` unchanged. Maintains
a global **Lineage Graph `L=(V,E)`**: nodes = atomic claims (+ source, timestamp
metadata); edges = dependency relations (**`supports`, `contradicts`**); nodes
marked **confirmed** (trusted) vs **unverified** (excluded from trusted context).
Pipeline (4 stages):
1. **Representation / Decomposition** — split message `M` into atomic claims `{aₖ}`
   (factuality + faithfulness claims).
2. **Decision — tri-state screen** vs. confirmed lineage:
   - 🟢 **Green** = entailed by confirmed → released & confirmed.
   - 🔴 **Red** = contradicts confirmed → **blocked, rollback evidence attached**.
   - 🟡 **Yellow** = neither → routed by policy.
3. **Policy routing** for Yellow (budget/risk): **Low-Intervention** (skip, tag
   unverified) / **Balanced** (verify Yellow from *hub* roles — aggregators,
   summarizers, decision-makers) / **Strict** (verify all). Then **comprehensive
   verification + risk arbitration** (external retrieval + LLM adjudication) →
   Verified-true→Green, Verified-false→Red+rollback, Unresolved→Yellow+risk tag.
   *Only confirmed increments become trusted downstream context.*
4. **Actuation — assembly + rollback**: if any Red (`Q_return≠∅`) → **block + feedback
   package** (rejected atoms, conflict evidence, rewrite directive) — intervention
   localized at the **atomic** level. Retries capped at `K`; **circuit breaker**
   isolates persistent offenders (persistent Red excluded; persistent Yellow
   forwarded high-risk-tagged but kept out of confirmed lineage).
- **Online** (inline, real-time block/rollback) OR **Offline** (replay historical
  logs → forensic **root-cause attribution**: identify root/high-degree corrupting
  nodes, trace propagation across agents & time, reconstruct `S(t)`).

## Key empirical findings (§VII)
- **Detection alone does NOT contain propagation.** Ablation `no_blocking` → BICR
  3.1% (≈ no-defense 2.2%); full Strict → 94%. **Rollback/isolation is essential.**
- `no_atomization` → high variance (atomization makes enforcement targeted).
- Operating points trade safety vs. cost: **Speed** (cost-aware, ~0.89 BICR),
  **Balanced** (0.93), **Strict** (0.94, highest token/latency).
- Agent self-reflection and prior guardrails (AGrail, CFG) reduce final infection
  in places but **collapse usable Safe-Completion** — governance keeps the workflow
  usable while containing.

## Stated design goals (directly reusable as Greenwood requirements)
- **G1** mitigate propagation of unverified claims into shared context.
- **G2** facilitate **early correction to prevent consensus lock-in** (correct
  before the belief hardens).
- **G3** strategic allocation of (expensive) verification budget toward high-impact
  (hub) positions & high-risk claims.
- **G4** **auditability & reproducibility** — traceable record of claim entry,
  propagation, correction.

## Limitations (author-stated)
Binary adoption observable misses partial internalization / semantic drift;
`(β,δ)` and `A` treated as stationary; `R` is a theoretical heuristic, not a
deployment-grade calibrated risk predictor; injection study is a controlled probe,
not full adversarial characterization (no tool-tampering / profile-poisoning).

## Implications for Greenwood (my synthesis)
- Validates Terra's requirement: a flat event stream (Chattermax = XMPP→Kafka→S3,
  temporal order) is **necessary but not sufficient**. Need a **first-class
  lineage/provenance DAG** (`supports`/`contradicts`, confirmed/unverified) coupled
  to the log. See `topics/01-provenance-vs-event-stream.md`.
- **Pruning requires action, not just observation** — block + rollback + circuit
  breaker, at **atomic-claim** granularity (finer than Chattermax's 12 message
  types). See `topics/03` and `topics/04`.
- Governance is **message-path middleware that leaves topology untouched** — squares
  with the garden's "transport, not merge engine" if framed as screen/annotate/block,
  not rewire. Open Q: in-bus vs. projection. See `topics/02`.
- Spectral/hub insight → budget verification by node centrality; the bus should
  expose topology/centrality so high-impact branches get more scrutiny.
