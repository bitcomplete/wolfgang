# Human–Computer Interfaces for Agents

**Date:** 2026-07-29
**Scope:** How humans supervise, steer, and trust agent systems — and what interface surface the software garden's late-July release minimally needs. Frame throughout: the first user is a **junior engineer running runtime operations for 4–5 apps** on Bit Complete's bc-prod/kploy platform.
**Method:** Synthesis of the discovery corpus (paths cited inline; index at `/Users/terra/Developer/wolfgang/research/software-garden-2026-07/00-research-summary.md`) plus targeted web research on interruption design, trust calibration, span-of-control, and current agent-supervision UI patterns.

---

## 1. The core problem

The release inverts the usual ops relationship: instead of a human doing operations with tools, **agents do operations and a human supervises**. That makes the interface the place where safety, trust, and economics are actually enacted — every guardrail in the design corpus (risk tiers, approval gates, cost caps, lineage) ultimately terminates in a screen, a Slack message, or a page that a specific junior engineer must correctly read and act on.

Four facts make this hard, all established in the corpus:

1. **The supervisor is the predicted weak point.** Juniors accept agent output readily, sometimes without comprehension (arXiv 2602.00496), passive AI use erodes exactly the debugging judgment supervision requires (Anthropic skill-formation study), and even seniors misjudge agent effectiveness (METR: 19% slower while believing 20% faster). Sources: `/Users/terra/Developer/wolfgang/research/software-garden-2026-07/notes/research-arxiv-multiagent-sdlc.md` §5.1, `/Users/terra/Developer/wolfgang/research/software-garden-2026-07/notes/research-practitioner-lessons.md` §4.
2. **Human attention is the binding constraint.** Machine-initiated work scales faster than human sensemaking; the AIDev mining study (456k agent PRs) found the human review bottleneck, not agent throughput, is what binds (`research-arxiv-multiagent-sdlc.md` §2.3, §5). Approval fatigue is a documented security bug, not a UX nuisance (`/Users/terra/Developer/wolfgang/research/software-garden-2026-07/notes/research-emergent-2.md` §7).
3. **Trust is asymmetric and fragile.** Only 8% of 696 surveyed practitioners run operational agents in prod; 60% cite trust as the #1 blocker; false positives destroy adoption faster than anything else (`research-practitioner-lessons.md` §3). A junior who gets three bogus agent findings in week one stops using the system.
4. **There is no budget for a new interface.** Bit Complete is a low-teens-headcount consultancy with no platform team; the junior already lives in Slack, pin, bithub, Grafana, kploy, and a terminal (`/Users/terra/Developer/wolfgang/research/software-garden-2026-07/notes/bitcomplete.md` §4, §6.5). The July surface must be composed from those, not invented.

The Wolfgang KB currently contains **no operator UI/UX, runbook, alerting, or day-2-ops design at all** — observability is one M6 ticket (T8.4) (`/Users/terra/Developer/wolfgang/research/software-garden-2026-07/notes/kb-wolfgang.md` §6). This report is therefore filling a genuine blank, not refining an existing design.

---

## 2. What the evidence says

### 2.1 Attention and interruption design — when should an agent page a human?

**The classic answer from ops practice:** a page (synchronous interrupt) is reserved for things that are *urgent, actionable, and real*; everything else goes to an asynchronous queue or a log. This is Google SRE alerting doctrine, and it transfers directly: most of what agents produce is neither urgent nor human-actionable and must never interrupt.

**The channel taxonomy that falls out of the corpus** — three delivery classes, mapped to the risk tiers already validated in `research-emergent-2.md` §1/§10:

| Channel | Semantics | What goes there (tier mapping) |
|---|---|---|
| **Page** (phone/SMS, ack required) | Drop what you're doing | App down / SLO breach (existing monitoring, not agent-invented); an agent blocked mid-**incident** needing a decision; break-glass events; budget hard-stop on a production-relevant agent |
| **Inbox** (Slack/dashboard queue, respond within hours) | A decision is needed, with evidence attached | T3 approvals; T4 two-person requests; patrol findings above confidence threshold; "agent gave up / escalating" |
| **Feed/digest** (activity stream, daily summary) | FYI — no action | T1 reads, T2 auto-actions with their logs, cost roll-ups, close-out notes |

**Quantitative anchors from current practice:**

- Enterprises running agent fleets target a **10–15% escalation rate** — the agent pauses for a human on roughly one in seven to ten tasks ([digitalapplied.com escalation design guide](https://www.digitalapplied.com/blog/human-in-the-loop-escalation-design-ai-agents-2026)). Below that, oversight is theater; far above it, the human is doing the work.
- Anthropic's trustworthy-agents research: a well-trained agent should **raise its own check-in rate as task difficulty climbs** — self-escalation frequency is a more honest signal than a verbalized confidence score (via [digitalapplied](https://www.digitalapplied.com/blog/human-in-the-loop-escalation-design-ai-agents-2026)).
- The approval-fatigue literature converges on **DENY rules → ALLOW rules → human residual**, with the junior's queue target at "a handful per day across 4–5 apps" (`research-emergent-2.md` §7, §10.6). If every low-risk action demands confirmation, reviewers are trained to approve reflexively and consequential decisions drown ([channel.tel interrupt patterns](https://www.channel.tel/blog/agent-interrupt-checkpoint-approval-patterns)).
- Trust-building pattern from AIOps vendors: **start confidence thresholds high** so only high-confidence findings notify anyone, and lower them as track record accrues — false positives kill trust fastest (`research-practitioner-lessons.md` §3, LogicMonitor/Register).

**Interrupt mechanics.** A working interruption needs four parts: a gate that intercepts the flagged action, a checkpoint that preserves agent state, a notification that reaches the right person, and a resume path ([channel.tel](https://www.channel.tel/blog/agent-interrupt-checkpoint-approval-patterns)); LangGraph's `interrupt()` + checkpointer is the reference implementation and the durable-pause detail matters because approvals can take hours (`research-emergent-2.md` §6). **Greenwood's event-sourced fold-the-log design already provides the hard part** (durable pause/resume) as an architectural property (`kb-wolfgang.md` §3); the July work is the notification and resume *affordance*, not the substrate.

**The NeuBird pattern is the target on-call UX:** on alert, the responder arrives to "a document outlining the explanation of what's happening and either giving me a solution or telling me who should get involved" (`research-practitioner-lessons.md` §3, The Register). The page delivers a pre-built explanation, not a raw alarm — this compensates for exactly what a junior lacks (system-state intuition).

### 2.2 Trust calibration and verifiable output

**The foundational frame** is Lee & See (2004), "Trust in Automation: Designing for Appropriate Reliance": the design goal is **calibrated trust** — reliance matching actual reliability — and both over-reliance and under-reliance are failures the interface must prevent ([30-year longitudinal review](https://pmc.ncbi.nlm.nih.gov/articles/PMC12562135/), [systematic review on appropriate trust in human-AI interaction](https://arxiv.org/pdf/2311.06305)). For this release the junior-specific evidence shows **mis-calibration in both directions** — over-reliance *and* overly cautious avoidance (arXiv 2602.00496, N=10 qualitative; corrected 2026-07-29 from the earlier one-sided "over-trust" reading) — so the interface must genuinely calibrate: push down unwarranted reliance *and* make warranted confidence legible enough that the junior doesn't freeze.

Concrete, evidence-backed calibration mechanisms:

1. **Verifiable output over persuasive output.** Inline evidence — sources, test results, metric deltas, diffs — turns output from "trust me" into "check me" ([appropriate-reliance intervention studies](https://arxiv.org/pdf/2412.15584)). The corpus's version: never render "agent recommends X — approve?"; always render the evidence pack (`research-arxiv-multiagent-sdlc.md` §5.1). Grounded verification stabilizes multi-agent belief while ungrounded opinion destabilizes it (arXiv 2606.27409) — the same rule applied to the human: show logs and metrics, not agent self-assessment.
2. **Metadata-conditioned authority, not model confidence.** The system should compute freshness/verification status deterministically and stamp it into every rendered answer ("last verified 2026-06-03; source changed since"), degrading to warnings or refusal below thresholds. Deterministic freshness resolution beats LLM self-adjudication by +10.8 points like-for-like (FC-SH; +28 vs HippoRAG-v2 multi-hop only — precision-corrected 2026-07-29) (arXiv 2606.01435), and model-internal confidence provably misses stale-context failures (`/Users/terra/Developer/wolfgang/research/software-garden-2026-07/notes/research-emergent-1.md` §5). This is the direct fix for the observed mahdi failure (authoritative answers from 7-week-frozen state) — the single most dangerous junior-facing failure found in discovery.
3. **Track-record-legible trust tiers.** Progressive trust must be computed from the recorded event log (N clean runs of an action type) and *displayed* — not held as operator vibes (`research-arxiv-multiagent-sdlc.md` §5; graduated-oversight framework arXiv 2606.22484; ITIL standard-change promotion in `research-emergent-2.md` §3). The UI meaning: an approval card should say "this action type: 14 clean runs, 0 rollbacks, promoted to auto on 2026-08-XX pending Terra sign-off."
4. **Live legibility as a trust mechanism.** Efecto's strongest pattern is watch-then-intervene: humans see agents work in real time and can step in at any point — "for a junior engineer trusting agents with prod-adjacent operations, this watch-then-intervene loop is the trust mechanism to copy" (`/Users/terra/Developer/wolfgang/research/software-garden-2026-07/notes/competitor-efecto.md`). Full live streaming is deferrable, but its cheap approximation — a tail-able per-task transcript, which Trellis already keeps in `agent-logs/` — is not.
5. **Protect learning while calibrating.** Require a one-line close-out ("what happened / why") on incident close and stated reasons on high-risk approvals, feeding the KB — the explanations-as-mentorship channel that offsets de-skilling (`research-practitioner-lessons.md` §6.8, `research-arxiv-multiagent-sdlc.md` §5.1).

### 2.3 Dashboards vs. conversational interfaces vs. notification streams

The 2026 practitioner consensus is blunt: **chat-first UX fails for supervision** ([HatchWorks agent UX patterns](https://hatchworks.com/blog/ai-agents/agent-ux-patterns/), [agentic-design.ai UI patterns](https://agentic-design.ai/patterns/ui-ux-patterns)). Supervision needs persistent, glanceable state and control affordances — start/stop, approvals, receipts, logs, rollback — that a linear conversation cannot carry. "Mission control" dashboards emerged as the named pattern for exactly this ([Ralph TUI](https://www.verdent.ai/guides/ralph-tui-ai-agent-dashboard), [Magentic-UI](https://arxiv.org/pdf/2507.22358), [orchestrator-pattern essay](https://zackproser.com/blog/orchestrator-pattern)).

The academically named version is SASE's **ACE (Agent Command Environment)**: a workbench where the human supervises and mentors agent teams by consuming **Merge-Readiness Packs** (what/why/evidence/tests/rollback) and **Consultation Request Packs** (agent-initiated escalations) — "the July release's operator surface *is* an ACE in miniature" (`research-arxiv-multiagent-sdlc.md` §2.3).

Each surface has a distinct, non-substitutable job:

- **Dashboard (pull, glanceable):** answers "is everything okay?" and "is this normal?" — per-app health, agent activity, pending decisions, and cost-vs-baseline ("today: $9 — typical: $6–14" beats a raw number; `/Users/terra/Developer/wolfgang/research/software-garden-2026-07/notes/research-emergent-3.md` §6). Trellis is the in-house precedent and vocabulary source: idea cards with status badges, progress rails, per-item dollar cost, iteration counts, Pool/Costs/Activity pages (`/Users/terra/Developer/wolfgang/research/software-garden-2026-07/notes/repo-trellis.md` UI section). Its gap is also instructive: single-user, no auth story — a shared operator dashboard needs at least a reverse proxy + network protection.
- **Notification stream (push):** carries the inbox and feed classes of §2.1. Approvals must arrive **where the junior already lives** — Slack buttons with evidence packs attached, the Policygenius/Teleport ChatOps pattern (`research-emergent-2.md` §6). Trellis's Telegram/web approval channel is the same shape.
- **Conversational interface (bidirectional):** the right tool for *investigation and explanation* — "why did this deploy roll back?", "what changed on app X yesterday?" — because diagnosis is genuinely dialogic. It is the wrong tool for control. Two hard requirements from the corpus: answers must carry provenance/age stamps (§2.2), and anything recomputable (git status, deploy versions, pod state) must be **derived live at answer time, never quoted from a stored index** (`research-emergent-1.md` implications 2–3). Trellis's runbook style — diagnostic command blocks plus copy-paste agent prompts, one artifact serving human and agent — is the model (`repo-trellis.md` implications 1).
- **CLI:** every UI affordance needs a CLI counterpart (Trellis precedent: Retry button + `trellis retry <idea> <role>`), because during an incident the terminal is where the junior will be.

A useful negative example: Buzz (Block) bets everything on one conversational surface (chat with agents as channel members) and ships **no approval gates, no fine-grained authorization, and nothing for operations** — reinforcing that a chat substrate alone cannot be an ops supervision surface (`/Users/terra/Developer/wolfgang/research/software-garden-2026-07/notes/competitor-hive.md` §7).

### 2.4 Approval and permission UX

The corpus already settles the *policy* (4-tier matrix, versioned policy-as-code at the tool-call boundary — `research-emergent-2.md` §10.1). The HCI findings are about how the gate looks and behaves:

1. **Name the human's role per tier.** The Knight Institute autonomy levels name levels by the *user's role*: operator → collaborator → consultant → **approver** → observer ([arXiv 2506.12469](https://arxiv.org/abs/2506.12469)). The junior is an **approver** for T3 and an **observer** for T1/T2 — which cleanly splits the UI into an approval inbox (decisions) and an activity feed (awareness). Mixing them is what produces fatigue.
2. **The approval card is the product.** Minimum contents, converged across OWASP / SASE / blast-radius vendors: what (exact action + normalized parameters), why (triggering evidence), blast radius, rollback plan, the agent's track record on this action type, and an expiry. Approval is **bound to the exact action** (OWASP schema: actor, tool, target, params, timestamp, expiry, policy version) — never a session grant (`research-emergent-2.md` §1, §10.2).
3. **Approve plans, not diffs.** Plan-then-execute research says the reviewable artifact is the plan with success criteria; CSA L2 = approve plans/batches, agent executes within approved scope (`research-arxiv-multiagent-sdlc.md` §2.1, `research-emergent-2.md` §2). For a junior this is decisive: judging a plan ("canary deploy app X, auto-rollback on SLO breach") is feasible; judging a raw diff is not.
4. **Shrink what must be judged.** The approved unit for deploys is "canary + auto-rollback," which manufactures reversibility and demotes the action from T4 toward T2 (`research-emergent-2.md` §3, §10.4) — the interface expresses risk *reduction*, not just risk *routing*.
5. **Two-person rule rendered as escalation, not blocking.** T4 cards show "requires you + Terra"; one tap notifies the second approver. CSA: authorization authority scales with autonomy (`research-emergent-2.md` §2).
6. **Reject/edit/respond, not just approve/deny.** LangGraph's HITL vocabulary (approve / edit / reject / respond) is the right verb set — a junior who can only click "approve" is a rubber stamp (`research-emergent-2.md` §6).
7. **A break-glass path must exist in the UI.** ITIL's emergency-change class: during an incident the junior can invoke a logged, expiring mode that widens T2/T3 (never T4-alone), pages Terra, and forces post-incident review — otherwise the junior bypasses the whole system during the first real outage (`research-emergent-2.md` §10.8).
8. **A classifier stage before the human is the scaling lever, not the MVP.** Codex's `auto_review` (LLM screens approval requests, fails closed) is the first mainstream implementation; the corpus recommends it as a fast-follow (`research-emergent-2.md` §5, §10.6).

### 2.5 Span of control — how many agents/apps can one junior supervise?

The oldest quantitative literature here is human supervisory control of UAVs. **Fan-out** — the maximum number of semi-autonomous units one operator can supervise — is governed by two variables: **neglect tolerance** (how long a unit stays safe/effective while ignored) and **interaction time** (how long each intervention takes) ([Olsen/Goodrich fan-out line, via BYU](https://faculty.cs.byu.edu/~mike/mikeg/papers/SMC2010.pdf); [Cummings, single-operator multiple-UAV architecture](http://www.dodccrp.org/files/IC2J_v1n2_01_Cummings.pdf); [Cummings, human performance in supervisory control](https://www.researchgate.net/profile/Missy-Cummings/publication/246399496_Human_Performance_Issues_in_Supervisory_Control_of_Autonomous_Airborne_Vehicles/links/61d8f9a2b8305f7c4b2c563a/Human-Performance-Issues-in-Supervisory-Control-of-Autonomous-Airborne-Vehicles.pdf)). Two durable findings transfer:

- **Fan-out is engineered, not discovered.** Raising autonomy raises fan-out — but at the cost of situation awareness and complacency — the over-reliance half of the junior mis-calibration literature. You buy span of control with neglect tolerance (sandboxes, caps, gates that make an ignored agent harmless) and cheap interactions (evidence packs that make each decision fast).
- **Single-operator supervision of ~4 moderately autonomous units was the practical envelope** in that literature — strikingly aligned with the release's 4–5 apps, but only under those two conditions.

The 2026 agent-era evidence is anecdotal but consistent: one coding agent is manageable, five becomes "branch soup" without coordination infrastructure ([parallel-workflow practitioner writeups](https://bobde-yagyesh.medium.com/the-parallel-ai-workflow-developer-setup-for-2026-5191f3d18e1a), [SSOJet parallel sub-agent survey](https://ssojet.com/blog/parallel-sub-agent-coding-tools)); structured setups with automated verification have run 12 sessions, but only with an orchestrator absorbing the coordination load. The AIDev finding (review bottleneck binds) and the oversight-under-load essay line (`research-arxiv-multiagent-sdlc.md` §5) say the same thing at fleet scale. Anthropic's 2026 trends report puts the industry answer as **layered oversight**: agents self-flag uncertainty, AI reviewers screen other agents, and humans receive only escalations.

**Verdict for this release:** one junior supervising 4–5 apps is *at the boundary* of the evidenced envelope, and feasible only if: T1/T2 never surface synchronously; the approval queue stays at a handful/day; each approval is decidable from its card in under ~2 minutes; every agent is harmless while ignored (caps + sandbox + gates); and there is exactly **one** inbox, not five per-app inboxes. The count that matters is not agents (patrols can be many) but **decision events per day reaching the human**.

---

## 3. Recommendation

**Design one supervision loop, not an application.** The July interface is: *one inbox + five app cards + evidence-packed approval cards + a paging policy* — composed entirely from surfaces the junior already uses (Slack, pin/bithub, Grafana, terminal). The unifying design rule, from the fan-out literature: **maximize neglect tolerance, minimize interaction time.** Every interface decision should be tested against those two variables.

Concretely:

1. **Slack is the notification and approval spine.** Approval cards (T3, T4-escalation, patrol findings, budget alerts) arrive as Slack messages with buttons; the daily digest is a scheduled Slack post. This is the ChatOps pattern with the strongest precedent and zero new infrastructure. Every approval action writes an OWASP-schema audit record to the event log.
2. **A per-app status card is the dashboard MVP** — health, last deploy + rollback button (kploy), agent activity last 24h, cost today vs. baseline, pending decisions, link to runbook. Rendered as a simple page on pin/bithub (the `bc-mdx-components` repo — "MDX components for agent-generated documents" — already exists for exactly this). Grafana stays the metrics deep-dive; the card is the junior's home screen. Skip any bespoke mission-control web app for July; Trellis's dashboard vocabulary is the model to grow into later.
3. **The conversational surface is the navigator, scoped to explanation** — with age/provenance stamps on every answer and derive-don't-quote enforced for runtime facts, per `research-emergent-1.md`. It never executes; it explains and drafts.
4. **Paging is scarce by policy:** app-down (existing monitoring), agent-blocked-during-incident, break-glass, budget hard-stop. Everything else is inbox or digest. Write this channel policy down as part of the release spec — it is one paragraph and it is the interruption design.
5. **Every stuck state gets both a button and a CLI command** (resume, retry, kill, break-glass) — the Trellis precedent, backed by Greenwood's fold-the-log resume.
6. **Defer the AI-review classifier layer, live streaming views, trust-promotion UI, and mobile push** — each is the right growth path (they raise fan-out later) but none is needed to supervise 4–5 apps at a handful of decisions/day.

This also is the differentiation story: Buzz has no approval UX, Efecto has no governance, Kasava has no ops surface. A junior-legible, evidence-packed approval loop **is** the product's visible face (`00-research-summary.md` §2).

---

## 4. What this means for the late-July release

### Must-have (the release is unsafe or unusable without these)

1. **Channel policy (page / inbox / digest) written into the spec**, mapped to the 4 risk tiers. One page of prose; governs everything else. *(Effort: hours.)*
2. **Slack approval cards with evidence packs**, bound to exact actions (what/why/blast-radius/rollback/track-record/expiry), verbs approve/deny/edit-request, durable pause under them (Greenwood resume), audit record on every decision. Two-person rendering for T4. *(This is the single highest-leverage build item.)*
3. **Queue discipline:** DENY → ALLOW → human-residual routing so the junior sees a handful of decisions/day. Without this, item 2 becomes rubber-stamp training. High initial confidence thresholds on patrol findings — three false positives in week one kills the system.
4. **Per-app status card × 5** on pin/bithub: health, last deploy + rollback affordance, agent activity, cost-today-vs-baseline tile (seeded with Anthropic's published anchors: ~$13/dev/active-day, 90% of users <$30/day), pending decisions. Glanceable in <30 seconds.
5. **Resume/retry/kill as first-class affordances** — button + CLI — for every agent task; a stuck-at-cap or crashed agent must be visible on the card, never silently dead.
6. **Break-glass mode** in the UI: logged, expiring, pages Terra, forces post-incident review. Must exist before the first real incident.
7. **Provenance and age stamps on every navigator answer** + derive-don't-quote for runtime facts. Days of work; fixes the most dangerous observed junior-facing failure mode.
8. **NeuBird-shaped incident pages:** when a page fires, it links to an agent-drafted explanation document (what's happening, recommended action, who to escalate to) — not a bare alert.

### Should-have (ship if time allows; cheap and compounding)

9. **Close-out notes:** one-line "what happened / why" required on incident close and reason-required on T3/T4 approvals, feeding the KB — the anti-de-skilling channel.
10. **Daily digest** (scheduled Slack post): per-app one-liners, spend, auto-actions taken, anything approaching thresholds.
11. **Track-record display on approval cards** ("this action type: N clean runs") backed by a plain standard-change register file — the promotion *mechanism* can be manual (Terra edits the file); only the display ships.

### Deferrable (post-release; the fan-out growth path, in priority order)

12. **Auto-review classifier** (Codex `auto_review`-style, fail-closed) screening approval requests before the human — the main lever for scaling past a handful of decisions/day.
13. **Mission-control web dashboard** consolidating the app cards, feed, and approval queue into one authenticated app (grow from Trellis's UI vocabulary; requires an auth story Trellis lacks).
14. **Live watch-then-intervene streaming** of agent transcripts (Efecto's trust mechanism); the cheap approximation — tail-able per-task logs — should exist from day one via the event log.
15. **Automated trust-tier promotion/demotion UI** over the standard-change register (demotion-on-incident automated first).
16. **Mobile push / paging integration** beyond whatever bc-prod monitoring already provides.

### The one-sentence spec

> The junior's entire supervisory world is: **one Slack inbox that only ever contains decisions worth making, five app cards that answer "is this normal?" at a glance, a navigator that explains with timestamps and never bluffs, and a page that only fires when a human is genuinely needed — with resume, rollback, and break-glass always one action away.**

---

## Sources

**Local corpus (absolute paths):**
- `/Users/terra/Developer/wolfgang/research/software-garden-2026-07/00-research-summary.md` — synthesis
- `/Users/terra/Developer/wolfgang/research/software-garden-2026-07/notes/kb-wolfgang.md` — Greenwood design state; absence of operator UX in KB
- `/Users/terra/Developer/wolfgang/research/software-garden-2026-07/notes/research-arxiv-multiagent-sdlc.md` — SASE/ACE, AIDev review bottleneck, junior over-trust, graduated oversight, SREGym
- `/Users/terra/Developer/wolfgang/research/software-garden-2026-07/notes/research-practitioner-lessons.md` — SRE survey, NeuBird pattern, false-positive discipline, METR, junior skill findings
- `/Users/terra/Developer/wolfgang/research/software-garden-2026-07/notes/research-emergent-2.md` — risk tiers, approval-record schema, autonomy levels, approval fatigue, ChatOps precedent, break-glass
- `/Users/terra/Developer/wolfgang/research/software-garden-2026-07/notes/research-emergent-3.md` — cost baselines, "is this normal?" dashboard, cutoff ladder
- `/Users/terra/Developer/wolfgang/research/software-garden-2026-07/notes/research-emergent-1.md` — staleness stamps, derive-don't-quote, metadata-conditioned authority
- `/Users/terra/Developer/wolfgang/research/software-garden-2026-07/notes/repo-trellis.md` — in-house dashboard/approval/runbook precedent
- `/Users/terra/Developer/wolfgang/research/software-garden-2026-07/notes/bitcomplete.md` — existing surfaces (Slack, pin, bithub, Grafana, kploy); operability bar
- `/Users/terra/Developer/wolfgang/research/software-garden-2026-07/notes/competitor-efecto.md` — watch-then-intervene legibility
- `/Users/terra/Developer/wolfgang/research/software-garden-2026-07/notes/competitor-hive.md` — Buzz: chat-only surface, no approval UX

**Web (this report's additional research):**
- [Cummings — Automation Architecture for Single Operator, Multiple UAV Command and Control](http://www.dodccrp.org/files/IC2J_v1n2_01_Cummings.pdf); [Human Performance Issues in Supervisory Control](https://www.researchgate.net/profile/Missy-Cummings/publication/246399496_Human_Performance_Issues_in_Supervisory_Control_of_Autonomous_Airborne_Vehicles/links/61d8f9a2b8305f7c4b2c563a/Human-Performance-Issues-in-Supervisory-Control-of-Autonomous-Airborne-Vehicles.pdf); [fan-out/neglect-tolerance line (BYU)](https://faculty.cs.byu.edu/~mike/mikeg/papers/SMC2010.pdf); [WPI HCI lab overview of human supervisory control](https://wp.wpi.edu/hcilab/human-supervisory-control/)
- [Trust-in-automation 30-year longitudinal review (Lee & See lineage)](https://pmc.ncbi.nlm.nih.gov/articles/PMC12562135/); [Systematic review: fostering appropriate trust in human-AI interaction](https://arxiv.org/pdf/2311.06305); [Evaluating interventions for appropriate reliance on LLMs](https://arxiv.org/pdf/2412.15584)
- [Levels of Autonomy for AI Agents (Knight Institute, arXiv 2506.12469)](https://arxiv.org/abs/2506.12469)
- [HatchWorks — Agent UX patterns (chat-first fails)](https://hatchworks.com/blog/ai-agents/agent-ux-patterns/); [agentic-design.ai UI/UX patterns](https://agentic-design.ai/patterns/ui-ux-patterns); [Ralph TUI mission-control dashboard](https://www.verdent.ai/guides/ralph-tui-ai-agent-dashboard); [Magentic-UI (arXiv 2507.22358)](https://arxiv.org/pdf/2507.22358); [Orchestrator pattern essay](https://zackproser.com/blog/orchestrator-pattern)
- [digitalapplied — HITL escalation design 2026 (10–15% escalation target; Anthropic self-escalation)](https://www.digitalapplied.com/blog/human-in-the-loop-escalation-design-ai-agents-2026); [channel.tel — agent interrupt/checkpoint/approval patterns](https://www.channel.tel/blog/agent-interrupt-checkpoint-approval-patterns); [waxell — approval workflows that scale](https://waxell.ai/blog/ai-agent-approval-workflows)
- [Parallel agent workflow practitioner writeups](https://bobde-yagyesh.medium.com/the-parallel-ai-workflow-developer-setup-for-2026-5191f3d18e1a); [SSOJet — parallel sub-agent tools](https://ssojet.com/blog/parallel-sub-agent-coding-tools); [Anthropic 2026 Agentic Coding Trends (via tessl.io)](https://tessl.io/blog/8-trends-shaping-software-engineering-in-2026-according-to-anthropics-agentic-coding-report/)

**Caveats:** the 10–15% escalation-rate figure and the "branch soup" span anecdotes are secondary/blog-grade — use as design heuristics, not citable facts. The UAV fan-out envelope (~4 units) is from a different domain; treat the *variables* (neglect tolerance, interaction time) as the transferable finding, not the number. Google SRE paging doctrine is cited from standard practice (SRE book ch. 6 / "My Philosophy on Alerting"), not re-fetched in this pass.
