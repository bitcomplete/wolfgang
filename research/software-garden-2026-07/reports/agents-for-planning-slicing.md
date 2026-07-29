# Using agents for planning and slicing

**Date:** 2026-07-29
**Purpose:** Focused report for the software-garden late-July 2026 release spec. Frame: first usable iteration ships end of July; first user is a **junior engineer running runtime operations of 4–5 apps** on Bit Complete's platform (bc-prod / kploy).
**Scope:** what research and practice say about agents doing planning, decomposition, and work-slicing — planner/executor splits, work-breakdown quality, keeping humans as deciders, failure modes (over-decomposition, plan drift, stale plans), and how wolfgang-style planning navigators should hand sliced work to implementing agents. Includes what a junior engineer needs so agent-produced plans are *checkable* rather than trusted blindly.
**Sources:** the research-thread notes cited by absolute path throughout (§ Sources), plus targeted web research done for this report (URLs inline).

---

## 1. The core problem

The garden already bets on agent-mediated planning in three places:

1. **Wolfgang itself** is a planning navigator whose whole job is turning discussion into decisions, specs, and work breakdowns (`/Users/terra/Developer/wolfgang/CLAUDE.md`). Its flagship output — the Greenwood work breakdown — is ~40 implementation-ready tickets across 10 projects and 6 milestones, explicitly designated as the source for Linear tickets (`notes/kb-wolfgang.md` §5).
2. **Mahdi/Thufir** encode a planner→spec-writer→implementer relay: mahdi holds workstreams and statuses, Thufir "translates Mahdi's product requirements into 1-shot technical specifications for implementers," and implementation agents report back via `update-mahdi` (`notes/repo-mahdi.md`).
3. **Greenwood's D7 decision** makes decomposition a *bus-side* responsibility: the bus, not the agents, authors the claim graph that governs work (`notes/kb-wolfgang.md` §3–4).

The problem for the release: **agent-produced plans are exactly the artifact the first user is least equipped to audit.** The junior-engineer literature predicts over-approval of agent output, sometimes without comprehension (arXiv:2602.00496, via `notes/research-arxiv-multiagent-sdlc.md` §5.1), and the garden's own navigator corpus already exhibits the two plan-specific failure modes that hurt a junior most — confident answers from 7-week-stale state, and snapshot plan documents that quietly become pseudo-state (`notes/repo-mahdi.md`). So the question is not "can agents plan?" (they can, with known caveats) but "what shape must agent-produced plans and slices take so that a junior can check them, and so that implementing agents can execute them without drift?"

---

## 2. What the evidence says

### 2.1 Planner/executor splits are the consensus production shape — with a verifier closing the loop

Multiple independent lines converge on separating planning from execution (`notes/research-arxiv-multiagent-sdlc.md` §2.1):

- **Plan-then-execute with role-scoped permissions.** PEAR (arXiv:2510.07505), "Architecting Resilient LLM Agents" (arXiv:2509.08646), Routine (arXiv:2507.14447), and the code-gen-agent survey (arXiv:2508.00083) converge: the planner emits explicit constraints and success criteria; the **executor runs under stricter tool permissions than the planner**; a verifier or verification pass closes the loop. Production systems (e.g. Manus) ship exactly this three-stage pipeline for traceability.
- **Security rationale, not just quality.** The plan-then-execute split is also a prompt-injection containment measure: the executor only follows the approved plan, so hostile content read mid-execution can't silently retarget the run — directly relevant given the practitioner finding that everything an agent reads (logs, issues, READMEs) is untrusted input (`notes/research-practitioner-lessons.md` §1, §5.6).
- **Practice has productized it.** Claude Code's Plan Mode (read-only explore → propose plan → human review → approve → execute → verify → report) is now the documented default for anything touching production state, >5 files, or hard-to-undo commands ([DataCamp plan-mode tutorial](https://www.datacamp.com/tutorial/claude-code-plan-mode), [claudefa.st planning modes](https://claudefa.st/blog/guide/mechanics/planning-modes)). By 2026 every major coding tool ships a spec/plan-first flavor — GitHub Spec Kit, AWS Kiro, Cursor, OpenSpec, BMAD — with Spec Kit's four-phase loop (**Specify → Plan → Tasks → Implement, each a markdown file the next phase reads**) as the canonical shape ([BCMS SDD guide](https://thebcms.com/blog/spec-driven-development), [Spec Kit walkthrough](https://matsen.fhcrc.org/general/2026/02/10/spec-kit-walkthrough.html)). Community claims of 60–80% fewer rework cycles are unverified marketing-grade numbers — cite the *pattern*, not the stat.
- **Verification must be grounded and early.** Delayed or ungrounded correction destabilizes multi-agent consensus into oscillation (arXiv:2606.27409); verifiers must be evidence-anchored (tests, logs, metrics) and placed early/at hotspots (`notes/research-arxiv-multiagent-sdlc.md` §3.2). Applied to planning: plan review is cheapest and most stabilizing *before* execution — which is also where the human gate naturally sits.

**Garden fit:** Greenwood's three-parties boundary (agent / verifier / dumb bus) and per-role least-privilege via per-message governance (D8) already match this shape; the wolfgang→implementer handoff should adopt the same split explicitly — wolfgang plans, implementing agents execute under narrower permissions, and the plan itself is the reviewable artifact.

### 2.2 Decomposition quality: conditional, bounded, and bus-side

- **Specification failure is a top-level failure category.** MAST (arXiv:2503.13657; 1,600+ annotated traces) puts "system design / specification issues" as one of three root failure buckets — under-specified or badly sliced tasks are a *primary* cause of multi-agent failure, not an implementation detail (`notes/research-arxiv-multiagent-sdlc.md` §1.1). This argues for structured task contracts on the bus rather than free-text handoffs.
- **Over-decomposition is a documented failure mode.** Sub-tasks that are too granular make plans overly long, add coordination overhead, and blow context: in tasks requiring more than ~10 actions agents lose track of completed steps; plans decomposed into dozens of sub-tasks exceed context and the agent forgets its own trajectory ([planning survey, arXiv:2402.02716](https://arxiv.org/pdf/2402.02716); [apxml task-decomposition guide](https://apxml.com/courses/agentic-llm-memory-architectures/chapter-4-complex-planning-tool-integration/task-decomposition-strategies)). Too-broad slices fail the other way: downstream agents get stuck with unfinishable work. Practitioner corroboration: plans touching 7+ files measurably degrade during execution as context fills ([claudefa.st](https://claudefa.st/blog/guide/mechanics/planning-modes)).
- **Decomposition should be conditional, decided by a cheap rubric.** The winning 2026 pattern (icat-agent, arXiv:2606.25514) is adaptive: a rubric-based quality check decides whether a task gets a single agent or a team, and whether to parallelize or explore first — beating fixed pipelines on SWE-bench while *reducing* cost (`notes/research-arxiv-multiagent-sdlc.md` §1.3). MAST + Anthropic/Cognition say multi-agent gains are often minimal and cost ~15× tokens — so slicing into parallel agent work is justified only for breadth-first or well-specified parallelizable tasks (§1.2).
- **Bus-side decomposition is externally validated.** D7 (bus decomposes claims; agents never author the graph that governs them) matches both icat-agent's message-passing-not-shared-context result and the governance-gap analysis (arXiv:2606.31498): decomposition and delegation constraints are precisely what agents can't be trusted to self-report and what MCP/A2A/ACP can't express (`notes/research-arxiv-multiagent-sdlc.md` §1.3, §4).

### 2.3 Work-breakdown quality: vertical slices, boring primitives, and the "go broad" anti-pattern

- **The best-documented agent slicing anti-pattern is horizontal breadth.** Böckeler's Fowler-site experience report names it directly: agents "go broad instead of incrementally implementing working slices" — wasted effort accumulates before flawed assumptions surface (`notes/research-practitioner-lessons.md` §2). The practitioner counter-pattern is **vertical slices**: each unit of work cuts UI-to-data and delivers one observable behavior end to end, small enough to fit an agent's context window; Addy Osmani's 2026 workflow is literally "spec → break into vertical slices small enough for the AI context window → implement sequentially" ([codemag AI-greenfield series](https://www.codemag.com/Blog/AIPractitioner/AIAGSD6), [timdeschryver workflow](https://timdeschryver.dev/blog/keep-agentic-ai-simple-a-practical-workflow-for-software-development)).
- **Slice boundaries should be boring SE primitives.** CAID (arXiv:2603.21489): workers in isolated git worktrees, integration via branch-and-merge with test-based verification, +25.6% on PaperBench — the *merge gate* is the coordination mechanism, and each slice's acceptance test is its checkability contract (`notes/research-arxiv-multiagent-sdlc.md` §1.4).
- **A slice is done when its evidence bundle says so.** SASE (arXiv:2509.06216) names the artifact: **Merge-Readiness Packs** — what/why/evidence/tests/rollback — as the unit a human adjudicates. AIDev (456k agent PRs) quantifies why: agent PRs are fast but lower-acceptance and structurally simpler; the binding constraint is the human review bottleneck, so slices must arrive pre-packaged for adjudication (`notes/research-arxiv-multiagent-sdlc.md` §2.3).
- **The garden already has a good ticket shape.** Wolfgang's WORK-BREAKDOWN tickets carry Why / Scope / Acceptance / Deps (`notes/kb-wolfgang.md` §5) — structurally close to a Merge-Readiness Pack's front half. What's missing per the evidence: an explicit **verification/evidence clause** (how the implementer proves acceptance, mechanically) and a **rollback/compensation clause** (which Greenwood's effect-gate design already requires for reversible effects — the ticket schema should mirror it).

### 2.4 Keeping humans as deciders: approve plans, not diffs — and keep the queue small

- **The gate belongs at the plan.** The plan-then-execute literature's practical payoff for a junior operator: "the plan is a reviewable artifact with success criteria" is the natural human approval gate — **approve plans, not diffs** (`notes/research-arxiv-multiagent-sdlc.md` §2.1). A junior who cannot evaluate a 400-line diff *can* evaluate "these 3 steps, this success criterion, this rollback" — if the plan is written to be evaluated.
- **Even the most bullish vendors keep the merge gate human.** GitHub's own agentic workflow ends at a human-reviewed PR; 62% of surveyed SREs want co-pilot not autopilot; the field's working pattern is investigation/explanation agents with humans keeping judgment calls (`notes/research-practitioner-lessons.md` §2–3).
- **Oversight capacity is the scarce resource.** Machine-initiated work scales faster than human sensemaking (oversight-under-load essays; AIDev's measured review bottleneck). Approval fatigue is a documented security bug — the emergent-gates research sets the target at a handful of human approvals per day via DENY → ALLOW → human-residual routing (synthesis `00-research-summary.md` §4.11, `notes/research-emergent-2.md`). For planning this means: **plan-level approval replaces N step-level approvals** — the human approves a sliced plan once, then only tier-3/4 actions inside it re-surface.
- **Graduated oversight, earned per track record.** Kang (arXiv:2606.22484): tier review intensity by risk × agent maturity × output criticality; relax review only against recorded baseline performance. Combined with mahdi's agent trust profiles ("trustworthy with clear specs, struggles with ambiguity"), trust tiers should come from the event log, not operator vibes (`notes/research-arxiv-multiagent-sdlc.md` §5; `notes/repo-mahdi.md`).

### 2.5 Failure modes: plan drift, stale plans, and plans-as-pseudo-state

Three related decay modes, all now documented in both research and practice:

1. **Plan drift during execution.** A plan's usefulness decays as the agent acts and the environment changes; agents without explicit replanning checkpoints "follow a wrong plan confidently, because each step is individually valid according to the plan." The fix is to treat the plan as **mutable state re-evaluated at checkpoints**, separating the plan from the execution log so decay is visible rather than silent ([Plan-and-Act, arXiv:2503.09572](https://arxiv.org/pdf/2503.09572); [agentspan on agentic planning](https://agentspan.ai/glossary/agentic-planning/); [agent-drift practitioner writeups](https://usewire.io/blog/agent-drift-why-long-running-ai-agents-lose-the-plot/)).
2. **The stale-plan-file problem.** In coding-agent practice: the agent and human agree a plan; reality changes during implementation; the code adapts but the plan file doesn't; on resume, **the stale plan becomes the source of truth again** ([Dutta, "The Stale /Plan Problem"](https://medium.com/@arijitdutta23/the-stale-plan-problem-in-coding-agents-cde2c741f8ab)). The prescription: the plan describes the work *as now understood* — when evidence invalidates the approach, the approach section changes; it is not a historical artifact of the first thought.
3. **Snapshot plans as pseudo-state — observed in the garden itself.** Mahdi's corpus accumulates point-in-time exploration/index documents that "don't decay gracefully and partially contradict newer statuses," while the navigator answers authoritatively from 7-week-stale statuses — and its own PROJECT-INDEX claims merge conflicts are resolved that are visibly unresolved in the working tree (`notes/repo-mahdi.md`). This is the stale-plan problem at navigator scale, and the freshness research gives the fix: metadata-conditioned staleness (two-date frontmatter `updated`/`verified`, age-stamped answers, derive-don't-quote anything recomputable) rather than model-judged freshness — deterministic freshness resolution beats LLM adjudication by ~+28 points (`scratchpad/notes/research-emergent-1.md`, arXiv:2606.01435, via synthesis §4.10).

Plus the granularity failures from §2.2 (over-decomposition → context loss and overhead; under-decomposition → unfinishable slices) and Böckeler's breadth-first waste (§2.3). A junior operator will not detect any of these unaided: each individual step of a drifted plan looks valid, and stale confidence reads as truth to someone without the experience to push back (`notes/repo-mahdi.md` "Hazard for a junior").

### 2.6 Handing sliced work to implementing agents: the handoff contract

What the evidence supports as the handoff shape from a wolfgang-style planning navigator to implementing agents:

- **Structured task contract, not free-text.** MAST's specification-failure bucket + the governance-gap paper argue each handed-off slice should be a typed artifact: goal, scope boundaries (in/out), acceptance criteria with a mechanical verification command, dependencies, risk tier of expected actions, budget, and rollback note. Wolfgang's Why/Scope/Acceptance/Deps ticket schema is 70% of this already (`notes/kb-wolfgang.md` §5).
- **Spawn briefs are the garden's native mechanism.** Under D8, spawn = `SPAWN_BRIEF` message + control event, and D2's graduated spawn-brief tiers + entailment guardrail survive as the pattern (`notes/kb-wolfgang.md` §3–4). The handoff contract above is simply *the schema of a spawn brief* — and because spawn briefs always screen under the policy cascade, the bus can enforce contract completeness mechanically (reject briefs without acceptance criteria or budget).
- **Isolated workspace + merge gate as the execution envelope.** Per CAID: each implementing agent gets an isolated worktree; integration is branch-and-merge behind test verification; the event log records delegations and merges (`notes/research-arxiv-multiagent-sdlc.md` §1.4).
- **Precedents in-house.** The mahdi→thufir→implementer relay (planner navigator → 1-shot technical spec → implementer, with `update-mahdi` reporting back and rolling last-3 communication logs) is the existing garden handoff pattern; Trellis's per-stage gating (`auto` / `human-review` / `llm-decides`), phase recommendations (`proceed/iterate/needs_review/kill`), and 3-iterations-forces-human-review cap are the existing execution-side pattern (`notes/repo-mahdi.md`, `notes/repo-trellis.md`). Both work; both are file/message conventions, not enforced contracts — Greenwood's contribution is making the contract screenable.
- **Report-back closes the loop.** The handoff isn't done at dispatch: the implementer returns a Merge-Readiness-Pack-style bundle against the same acceptance criteria, and the plan/status artifact is updated (the update-mahdi discipline — currently "under-exercised," which is exactly how mahdi went stale; `notes/repo-mahdi.md`).

### 2.7 What makes an agent-produced plan *checkable* by a junior

Synthesizing across the junior-specific findings (arXiv:2602.00496 over-approval; Anthropic skill-formation passive-use de-skilling; the METR perception gap; Böckeler's "juniors catch category 1, miss categories 2–3" — `notes/research-practitioner-lessons.md` §4):

A plan is checkable when a junior can verify it **without reconstructing context or exercising judgment they don't yet have**:

1. **Falsifiable acceptance criteria per slice** — a command to run or metric to read, not "works correctly." Checking becomes mechanical.
2. **Explicit scope boundaries** ("does NOT touch X") — the junior can spot violations without understanding the domain.
3. **Risk-tier labels on every step** — the plan itself declares which steps are T1/T2 (auto) vs T3/T4 (gated), so the junior knows where their decision authority actually engages (`notes/research-emergent-2.md` tiering, via synthesis §6.3).
4. **Evidence attached, not asserted** — approval cards show test results, metric deltas, diffs; never "agent recommends X — approve?" (arXiv survey §5.1 countermeasures).
5. **Freshness stamps in the artifact** — plan sections carry `updated`/`verified` dates; anything recomputable (git status, deploy versions) is derived at read time, never quoted (`research-emergent-1.md`).
6. **Small enough to hold in one head** — the vertical-slice bound (one observable behavior, single-digit files) doubles as the junior-comprehension bound.
7. **A stated reason required on high-risk approvals** — forces engagement over rubber-stamping, and the required one-line close-out note feeds the KB, making review a learning channel rather than a de-skilling one (`notes/research-practitioner-lessons.md` §6.8).

---

## 3. Recommendations

**R1. Adopt plan-then-execute as the garden's normative agent workflow, with the plan as the human gate.** Wolfgang (and any planning navigator) produces plans; implementing agents execute them under narrower permissions; the junior approves *plans* (once, with tier-labeled steps) and only tier-3/4 actions re-surface mid-execution. This is the highest-leverage way to reconcile "human as decider" with "handful of approvals per day."

**R2. Standardize a slice contract — the spawn-brief schema.** Extend the WORK-BREAKDOWN ticket shape (Why/Scope/Acceptance/Deps) into the universal handoff artifact: add *verification command*, *risk tier*, *budget*, and *rollback/compensation note*. Use it for wolfgang→implementer handoffs now (as markdown convention) and as the `SPAWN_BRIEF` payload schema in Greenwood later, where the bus rejects incomplete briefs mechanically.

**R3. Slice vertically, decompose conditionally.** Default to a single strong agent per slice; each slice = one observable behavior, ≤ single-digit files, with its own acceptance test; parallelize only breadth-first read-only work (multi-app triage) or rubric-certified well-specified tasks. Cap plan length — a plan needing dozens of sub-tasks is a signal to re-scope, not to spawn more agents.

**R4. Treat plans as mutable state with mandatory refresh points.** Separate plan from execution log; re-evaluate the plan at checkpoints (slice boundaries are the natural ones); when evidence invalidates the approach, the plan file changes. Apply the freshness discipline (two-date frontmatter, age-stamped rendering, derive-don't-quote) to plans, statuses, and work breakdowns alike — a plan is just another KB entry that goes stale.

**R5. Close the loop or the planner goes blind.** Require implementers to report back against the brief's acceptance criteria (Merge-Readiness-Pack shape: what/why/evidence/tests/rollback) and require the status/plan artifact update as part of "done." Mahdi's staleness is the measured cost of leaving this to willpower.

**R6. Design plan-review as the junior's mentorship channel.** Evidence-rich plan cards, reasons required on high-risk approvals, one-line close-out notes feeding the KB. The literature says the junior will over-approve; the design must make blind approval harder than informed approval.

---

## 4. What this means for the late-July release

Frame: the July ship is realistically Greenwood-M1-scope plus a pulled-forward ops/governance layer (per synthesis §6). Planning-and-slicing items below are ranked accordingly.

### Must-have (July)

1. **The slice-contract template, as a checked-in markdown convention.** One file schema for every piece of work handed to an implementing agent: Why / Scope (in AND out) / Acceptance (with a runnable verification command) / Deps / Risk tier / Budget / Rollback note. Zero infrastructure — it's a template plus doctrine — and it fixes MAST's #1 failure bucket at the handoff. Apply it retroactively to the ~40 Greenwood tickets before any Linear import (which still awaits Terra's authorization).
2. **Plan-approval as the junior's primary gate.** The operator UX for any multi-step agent work: agent proposes a tier-labeled plan with evidence; junior approves the plan; only T3/T4 steps re-prompt. Never step-by-step approval spam (fatigue) and never plan-free autonomy (Replit lesson: instructions are not guardrails — the tier gates stay structural in the execution path even after plan approval).
3. **Checkable-plan rules baked into wolfgang's navigator doctrine.** Wolfgang must emit plans that carry falsifiable acceptance criteria, explicit out-of-scope lines, and freshness stamps; and must render last-verified dates in answers, softening authority when stale. Days of work (doctrine + templates), and it neutralizes the observed mahdi hazard before a junior ever queries a navigator.
4. **Vertical-slice + single-agent-default doctrine, written into the release spec.** One sentence of policy with outsized effect: one agent per slice, one observable behavior per slice, parallel fan-out only for read-only breadth-first investigation. Prevents both the 15× token multiplier and the "go broad" waste on a bootstrapped budget.
5. **Report-back required for "done."** Close-out = evidence bundle + status/plan update + one-line note to the KB. This is what keeps the July system's plans from being stale by August.

### Deferrable (post-July)

6. **Bus-enforced brief screening** (reject incomplete spawn briefs mechanically) — lands with Greenwood M2/M3 per-message governance; the markdown convention covers July.
7. **Scheduled replanning checkpoints / plan-decay automation** (auto-flag plans whose `verified` date lags execution events) — valuable, but manual checkpoint-at-slice-boundary discipline suffices at 4–5 apps scale.
8. **Track-record-based trust promotion for planners/implementers** (autonomy earned per action-type from event-log evidence) — needs an accumulated log first; ship at CSA-L2-equivalent fixed tiers in July.
9. **Isolated-worktree + merge-gate automation (CAID-style)** for parallel implementers — July's single-agent default makes this mostly moot; adopt when parallel implementation actually starts.
10. **Rubric-based decomposition triage** (cheap classifier deciding single-agent vs team, à la icat-agent / D7) — Greenwood-M2+ material; in July the human is the rubric.

### One recorded risk

The July release's planning surface is **wolfgang itself** — and the KB currently lacks the release definition (no July date, no persona, no 4–5-apps scenario; `notes/kb-wolfgang.md` §6). A planning navigator that can't see its own release plan will slice against the wrong goal. Writing the release into the KB is a precondition for every item above.

---

## Sources

### Notes files (this research thread)
- `/Users/terra/Developer/wolfgang/research/software-garden-2026-07/00-research-summary.md` — synthesis
- `/Users/terra/Developer/wolfgang/research/software-garden-2026-07/notes/kb-wolfgang.md` — Greenwood design state, D7/D8, WORK-BREAKDOWN ticket shape
- `/Users/terra/Developer/wolfgang/research/software-garden-2026-07/notes/research-arxiv-multiagent-sdlc.md` — MAST, PEAR/plan-then-execute, CAID, icat-agent, SASE/AIDev, junior studies
- `/Users/terra/Developer/wolfgang/research/software-garden-2026-07/notes/research-practitioner-lessons.md` — Böckeler/Fowler, Replit, SRE survey, junior learning debt
- `/Users/terra/Developer/wolfgang/research/software-garden-2026-07/notes/repo-mahdi.md` — mahdi→thufir handoff, trust profiles, staleness/pseudo-state anti-patterns
- `/Users/terra/Developer/wolfgang/research/software-garden-2026-07/notes/repo-trellis.md` — per-stage gating, phase recommendations, iteration caps
- `/Users/terra/Developer/wolfgang/research/software-garden-2026-07/notes/competitor-kasava.md` — planning-platform positioning lesson
- `/Users/terra/Developer/wolfgang/research/software-garden-2026-07/notes/research-emergent-2.md` — risk tiers, approval-fatigue routing (via synthesis)
- `/Users/terra/Developer/wolfgang/research/software-garden-2026-07/notes/research-emergent-1.md` — freshness/staleness discipline (via synthesis)

### Web (fetched/searched for this report, 2026-07-29)
- Spec-driven development / GitHub Spec Kit: https://thebcms.com/blog/spec-driven-development ; https://matsen.fhcrc.org/general/2026/02/10/spec-kit-walkthrough.html ; https://www.fundesk.io/spec-driven-development-github-spec-kit-guide
- Plan Mode practice: https://www.datacamp.com/tutorial/claude-code-plan-mode ; https://claudefa.st/blog/guide/mechanics/planning-modes ; https://baeseokjae.github.io/posts/claude-code-plan-mode-guide-2026/
- Vertical slicing with agents: https://www.codemag.com/Blog/AIPractitioner/AIAGSD6 ; https://timdeschryver.dev/blog/keep-agentic-ai-simple-a-practical-workflow-for-software-development ; https://jeremydmiller.com/2026/06/04/the-codebase-is-the-prompt-wolverine-vertical-slices-and-ai-assisted-development/
- Decomposition granularity / planning survey: https://arxiv.org/pdf/2402.02716 ; https://apxml.com/courses/agentic-llm-memory-architectures/chapter-4-complex-planning-tool-integration/task-decomposition-strategies
- Plan drift / stale plans: https://medium.com/@arijitdutta23/the-stale-plan-problem-in-coding-agents-cde2c741f8ab ; https://arxiv.org/pdf/2503.09572 (Plan-and-Act) ; https://agentspan.ai/glossary/agentic-planning/ ; https://usewire.io/blog/agent-drift-why-long-running-ai-agents-lose-the-plot/
- Key papers via arXiv notes: MAST arXiv:2503.13657 ; CAID arXiv:2603.21489 ; icat-agent arXiv:2606.25514 ; SASE arXiv:2509.06216 ; AIDev arXiv:2507.15003 ; junior agency arXiv:2602.00496 ; graduated oversight arXiv:2606.22484 ; governance gaps arXiv:2606.31498 ; delayed verification arXiv:2606.27409

**Evidence-quality caveats:** Spec Kit rework-reduction percentages are community/marketing grade — pattern only. The stale-plan Medium piece and agent-drift blogs are practitioner essays (low formal evidence, but consistent with Plan-and-Act and the garden's own observed mahdi failure). Several arXiv figures inherited from the survey note were read from abstracts, not full PDFs.
