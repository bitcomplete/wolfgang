# Wolfgang KB Research Notes — Software Garden & Agent Bus (Greenwood)

Date: 2026-07-29
Scope: full read of `/Users/terra/Developer/wolfgang` — `knowledge/`, `research/`, `config.json`, `README.md`, `CLAUDE.md`.
Purpose: inform the software-garden release spec (first usable iteration end of July 2026; first user is a JUNIOR engineer running runtime operations for 4–5 apps).

---

## 1. What Wolfgang is

- Wolfgang is a **project-planning and ideation navigator** (Autonav framework, `config.json` v0.1.0, created 2026-06-30). It is a *thinking partner* KB, not a code repo: it pressure-tests ideas, records decisions with rationale, and turns discussion into specs and work breakdowns (`CLAUDE.md`, `README.md`).
- Config description: planning navigator "for the software garden … across projects like Chattermax, Chalet, and Chizu" (`config.json`, `knowledge/agents/navigators/wolfgang.md`).
- Its flagship (and currently only) design work is **Greenwood** — the "Agent Bus" — filling `knowledge/ARCHITECTURE.md` and all of `research/`.

## 2. What "the software garden" means in this KB

- Per `research/README.md` (prior-art note): "the software garden" is **a family of earlier internal Bit Complete prototypes** — named *Chattermax* (XMPP-based agent chat bus), *Chalet*, *Chizu* (automated refinement/eval loop), *Gardener* (agent-as-seed lifecycle: `SeedPlanted/SeedForked/SeedRewound/SeedPromoted` JSONL journal), plus *Bitmap* (BIT-8 vocabulary) and *Hotspots/ASH* ("soil testing lab").
- In this KB the garden appears **purely as prior art and lessons learned**; "Greenwood is a fresh design with no dependency on them" (`research/README.md`).
- Documented garden lessons Greenwood explicitly corrects (topics 01–05):
  - **Temporal-only event model, no lineage/causal DAG** — errors can't be traced or pruned (`topics/01`).
  - **Three unreconciled event vocabularies, no version field, no migration plan** — the "no-version regret"; Greenwood answers with one Protobuf envelope + Schema Registry (`topics/03`, T1.1).
  - **Message-as-atom** — no sub-message claim granularity; Greenwood's atomic claim is genuinely new (`topics/03`).
  - **No faulty-branch lifecycle** — garden could fan-out/freeze/resurrect healthy branches but not detect/kill/rewind faulty ones; Gardener's seed vocabulary is the model to borrow (`topics/04`, T7.1).
  - **Chattermax lost ordering** (cited in T1.2 as the reason keyed partitioning is load-bearing).
  - **Unsolved garden ops problems carried forward as design targets** (topic 08 §7 / P8): sandboxing, rate limiting, cost control, crash/failure detection (garden open Q#7), multi-agent test harness.
- NOTE: the KB does **not** define the *new* "software garden" ecosystem vision (the thing shipping in July). Here "garden" = the legacy prototype family. The new ecosystem framing must come from elsewhere (or be written into this KB).

## 3. The Agent Bus: Greenwood — current design state

Source of truth: `knowledge/ARCHITECTURE.md` (top-level), `research/decisions.md` (P0, D1–D8), `research/topics/08-concrete-spec.md` (buildable spec), `research/NAMES.md`.

**One-liner:** an event-sourced agent runtime + coordination bus on Kafka. Every agent action is an immutable event; all state is a projection (fold) of the log. Founding constraint: multi-agent systems fail by **error cascade → false consensus** (grounded in *From Spark to Fire*, Xie et al. 2026, arXiv:2603.04474, reviewed in full in `research/papers/spark-to-fire/`); the bus must preserve the **genealogy** of interactions so faulty branches can be traced and pruned.

**Components (arboreal naming, `NAMES.md`):**
- **Greenwood** — the whole system (bus + runtime).
- **Annals** — immutable Kafka event log, keyed by `lineage_root` (per-branch total order), tiered hot→S3. Log = transport; no separate messaging layer.
- **Rootlines** — lineage DAG projection (claims as nodes, provenance edges; `descendants()` = rollback blast radius).
- **Grieve** — governance/verifier: per-message policy cascade, trust transitions, `confirmed` projection, circuit breaker.
- **Coldframe** — evals: feedback events → frozen-context replay evals → hermetic reproduce → tweak harness → regression suite.
- **Graft** — harness adapters (Claude Code, Codex, pi.dev, Hermes) via Protobuf envelope + gRPC lifecycle (`init/step/snapshot/resume/stop`). Bare name "graft" is taken on all registries → namespace as `greenwood-graft`/`@greenwood/graft`.

**Key mechanics:**
- **Resume:** agent state is a deterministic fold of its events; any host rebuilds byte-identical prompt and hits the content-addressed LLM prompt cache (cache TTL is a per-agent cadence policy, 5-min default; cache continuity is an opportunistic latency win, not an economic pillar). Snapshots bound replay; content-hash `event_id` gives idempotent replay; session epochs fence zombie producers.
- **Messaging is the only inter-agent primitive (D8):** messages to an agent or room; inbox = projection of policy-permitted messages; spawn = `SPAWN_BRIEF` message + control event. Handoff was demoted from primitive to pattern.
- **Governance = per-message policy cascade, cheapest first (D7/D8):** static rules (no ML; acks deliver, spawn briefs + hub-bound always screen) → encoder triage (~ms, sampled audits make false-green a measured SLO) → bus-side decomposition to atomic claims + tri-state screen against `confirmed` → selective LLM verification. Only screened content enters trusted context. Intra-agent work is captured but only retroactively decomposed (forensics free because the log is immutable).
- **The claim is the atom:** unit of trust, provenance, rollback. Decomposition is bus-side (agents never author the graph that governs them); provenance is mechanical via per-turn `context_manifest`.
- **Rollback repairs belief, not the world** (P0 scope-honesty, 2026-07-02): compensating events, never deletes; **effect gate** (topic 08 §5a) requires confirmed premises before IRREVERSIBLE tool calls; REVERSIBLE needs a registered compensation handler.
- **Cost model** (topic 11 + live calculator `research/cost-model/calculator.html`): three buckets — state capture/replay (cheap: low tens of $k/yr @500 agents), eval inference (large), governance inference (the marginal cost; scales with communication, ~$22k/yr @500 agents at mixed topology, ~0.1% overhead vs D7's rejected $1.46M).

## 4. Decided vs. open

### Decided / confirmed (Terra)
- **P0 — event sourcing end-to-end** (Terra, 2026-06-30; FOUNDATIONAL). Trust, lifecycle, snapshots all events; state only ever a fold.
- **D7 — bus-side decomposition, mechanical provenance, classifier triage** (Terra, 2026-07-02).
- **D8 — messaging as the inter-agent primitive; governance per-message** (Terra, 2026-07-02) — the cost-driven re-scope; supersedes handoff-as-primitive.
- **Build on Kafka** (Terra, 2026-06-30, topic 05) — the Kafka *API*, not bound to Chattermax choices.
- Three-parties boundary (agent / verifier / dumb bus) — Terra confirmed under D1.
- Caching is NOT a design driver at handoff (Terra, D2).

### Proposed, pending Terra's final confirmation (decisions.md front-matter: "proposed decisions pending Terra's final confirmation")
- **D1** hybrid claim decomposition (partially superseded by D7 — core split stands).
- **D2** graduated spawn-brief tiers + entailment guardrail (survives as pattern per D8).
- **D3** async governance + boundary gate (now per-message per D8).
- **D4** hermetic replay evals, evals-as-events, statistical pass-rate.
- **D5** Graft protocol / harness-agnostic.
- **D6** backing store: Kafka API on object-store-native engine (AutoMQ preferred / WarpStream; raw-S3 fallback only).

### Explicitly open questions (recorded)
- **Runtime language/stack** (Kafka Streams JVM vs Rust/rdkafka) — "Blocks P1/P2 shape. Needs Terra." (`WORK-BREAKDOWN.md` open questions).
- **Lineage store:** compacted topic only vs external graph DB (Memgraph/Neo4j) — affects T4.2/T8.4.
- **Verifier's own trust model** — who verifies the verifier; escalation policy open.
- **Claim-state retention:** monotone claim keyspace breaks latest-per-key compaction at scale → pick TTL-after-terminal-state vs DB-backed projection **before M2** (D6, resolved-analysis note).
- **Measure real message rates per topology** — top open parameter of the D8 cost model.
- **Produce latency on the S3 path** (D6).
- Effect classes / compensation handlers recorded as their own design area (P0 scope note).
- Wolfgang navigator's own OUT-OF-SCOPE boundaries "not yet defined" (`knowledge/agents/navigators/wolfgang.md`).

## 5. Work breakdown & sequencing (`research/WORK-BREAKDOWN.md`)

- Status: **draft; SOURCE for Linear tickets — do NOT create in Linear until Terra authorizes** (repeated twice in the file; also in auto-memory).
- 10 projects (P1–P10), 6 milestones, ~40 implementation-ready tickets with Why/Scope/Acceptance/Deps.
- Critical path: **M1 single-agent MVP** (event spine + runtime + resume, one agent, no governance) → **M2 claims + governance** → **M3 messaging + per-message governance** (the differentiator) → **M4 evals (Coldframe)** / **M5 Graft adapters** → **M6 hardening** (sandboxing, rate limits, cost budgets, observability, sim harness).
- Model pinned in tickets: `claude-opus-4-8`, adaptive thinking, `cache_control` breakpoints, `ttl:"1h"` (T2.2).

## 6. July release & junior-engineer-operator scenario — what the KB says

**Nothing.** Verified by grep across all `.md` files: there is **no mention of a July 2026 release date, deadline, "first usable iteration," a junior engineer, an operator persona, or the 4–5-apps runtime-operations scenario anywhere in the KB.** The KB is a design/spec corpus with milestone *ordering* (M1–M6) but **no dates, staffing, or target-user definition**. The closest artifacts:

- Milestone sequencing implies the earliest coherent ship is **M1 (single-agent MVP: event spine, runtime, resume)** — everything else depends on it.
- P8/M6 hardening tickets (T8.1–T8.5: sandboxing, rate limiting, per-agent token budgets, observability dashboards, sim harness) are the items most relevant to an *operator*, and they are scheduled **last**.
- Observability is only a ticket (T8.4: "trace an outcome through the lineage DAG; dashboards over the event stream"); there is no operator UI/UX, runbook, alerting, or day-2-ops design anywhere.
- Ops problems the garden left unsolved (crash detection, runaway agents, cost control) are named design targets (topic 08 §7) but unbuilt.

### Implications for the release spec (my analysis, not KB content)
1. **Gap to record:** the July release goal and the junior-operator persona are absent from wolfgang; they should be written into the KB (decisions/open-questions) so the planning navigator can reason about them.
2. **Scope tension:** Greenwood's differentiator (M3 per-message governance) is 3 milestones deep; a July "first usable iteration" almost certainly means **M1-level scope** (single-agent, event-sourced, resumable) — the release spec should say which milestones are in/out.
3. **Operator-fit tension:** everything a junior operator needs day-to-day (dashboards, crash detection T7.3, rate limits, cost budgets, runbooks) sits in M3/M6 — the release spec must either pull minimal ops affordances forward or scope the operator's duties to what replay/logs give for free (the immutable log is itself a strong audit/debug substrate).
4. **Two blocking decisions gate any build start:** runtime language/stack (blocks P1/P2) and the D-log confirmations (D1–D6 still "proposed"). These need Terra before ticketing.
5. **Linear guardrail:** work breakdown is ready to become tickets verbatim but requires Terra's explicit authorization (also enforced in auto-memory).

## 7. Source map (all paths absolute)

| Topic | File |
|---|---|
| High-level architecture, diagrams, cost summary | `/Users/terra/Developer/wolfgang/knowledge/ARCHITECTURE.md` |
| Decisions log P0, D1–D8 | `/Users/terra/Developer/wolfgang/research/decisions.md` |
| Work breakdown (10 projects, M1–M6, ~40 tickets) | `/Users/terra/Developer/wolfgang/research/WORK-BREAKDOWN.md` |
| Component names + rationale | `/Users/terra/Developer/wolfgang/research/NAMES.md` |
| Research index + garden prior-art note | `/Users/terra/Developer/wolfgang/research/README.md` |
| Buildable spec (envelope, topics, folds, policy cascade) | `/Users/terra/Developer/wolfgang/research/topics/08-concrete-spec.md` |
| Cost/scalability model + calculator | `/Users/terra/Developer/wolfgang/research/topics/11-scalability-and-cost.md`, `/Users/terra/Developer/wolfgang/research/cost-model/calculator.html` |
| Garden prior-art analyses | `/Users/terra/Developer/wolfgang/research/topics/01…05` |
| Resume/caching design | `/Users/terra/Developer/wolfgang/research/topics/06-agent-resume-and-caching.md`, `07-runtime-design-resume-genealogy-handoff.md` |
| Evals | `/Users/terra/Developer/wolfgang/research/topics/09-agent-eval.md` |
| Graft adapters | `/Users/terra/Developer/wolfgang/research/topics/10-graft-harness-adapters.md` |
| Founding paper review | `/Users/terra/Developer/wolfgang/research/papers/spark-to-fire/spark-to-fire.review.md` |
| Navigator config/identity | `/Users/terra/Developer/wolfgang/config.json`, `/Users/terra/Developer/wolfgang/knowledge/agents/navigators/wolfgang.md`, `/Users/terra/Developer/wolfgang/CLAUDE.md` |
