# Reducing Cognitive Debt

**Report for the software-garden late-July 2026 release spec.**
Date: 2026-07-29. Author: research subagent (workflow-orchestrated).
Frame: first usable iteration ships end of July 2026; first user is a **junior engineer running runtime operations of 4–5 apps** on Bit Complete's bc-prod/kploy platform.

Scope: how the garden minimizes the mental overhead of understanding, operating, and evolving 4–5 apps — what cognitive debt is, how agent-heavy systems accumulate it, what cognitive load theory says platforms should do about it, and the concrete mechanisms the garden should ship.

---

## 1. The core problem

A junior engineer operating 4–5 apps with agent help faces a mental-overhead problem that is *worse*, not better, than classic ops — unless the platform is deliberately designed against it. Three distinct debts compound:

1. **Comprehension debt** — the growing gap between how much code/system exists and how much any human genuinely understands. Addy Osmani's definition: "the growing gap between how much code exists in your system and how much of it any human being genuinely understands" — and, critically, unlike technical debt it "accumulates invisibly without triggering friction signals" ([Osmani, Comprehension Debt](https://addyosmani.com/blog/comprehension-debt/); republished by [O'Reilly Radar](https://www.oreilly.com/radar/comprehension-debt-the-hidden-cost-of-ai-generated-code/)). Agents are prolific producers of artifacts nobody read: code, configs, dashboards, even documentation.

2. **Cognitive debt (team/individual)** — erosion of the *understanding itself*. Two converging strands:
   - **Team-level**: DX's framing — cognitive debt "lives in the brains of the developers," is the fragmentation of Peter Naur's "program as theory," and accumulates when velocity outpaces comprehension and rationale goes undocumented ([getdx.com](https://getdx.com/blog/cognitive-debt-the-hidden-risk-in-ai-driven-software-development/)). A 2026 "triple debt" model formalizes this: technical debt (code layer), cognitive debt (erosion of shared understanding), and **intent debt** (un-externalized goals, constraints, and rationale that both humans *and agents* need) ([arXiv 2603.22106](https://arxiv.org/pdf/2603.22106)).
   - **Individual-level**: the MIT Media Lab EEG study that popularized the term — "Your Brain on ChatGPT: Accumulation of Cognitive Debt when Using an AI Assistant" ([arXiv 2506.08872](https://arxiv.org/abs/2506.08872), [brainonllm.com](https://www.brainonllm.com/)) — found LLM-assisted writers showed the weakest neural connectivity, lowest ownership of their own output, and couldn't quote their own work. Cognitive debt there = repeated reliance on external systems replacing the effortful cognition needed for independent thinking.

3. **Context scatter** — the operational knowledge that *does* exist lives in the wrong places: chat transcripts, agent session logs, a maintainer's head, stale READMEs. The garden's own discovery pass documented every variant of this (see §2.3).

### Why agent-heavy systems accumulate it faster

- **Generation outpaces evaluation.** "A junior engineer can now generate code quicker than a senior engineer can audit it critically" (Osmani). The AIDev mining study (456k agent PRs) found the binding constraint is the *human review bottleneck*, not agent throughput (`notes/research-arxiv-multiagent-sdlc.md` §2.3).
- **Decisions happen in chats and vanish.** Agent sessions are where diagnosis, trade-offs, and fixes actually occur; without a capture mechanism, each session's understanding evaporates at session end. Most teams don't even capture full decision traces — "without full traces the post-mortem is speculation" (`notes/research-practitioner-lessons.md` §2, scaling-discipline synthesis).
- **Passive use de-skills the operator.** The Anthropic skill-formation RCT (52 engineers): AI-assisted participants scored **17 points lower on comprehension/debugging quizzes (50% vs 67%)**; passive code-generation delegation scored below 40%, while *conceptual-inquiry* use scored above 65% ([Osmani's summary](https://addyosmani.com/blog/comprehension-debt/); `notes/research-practitioner-lessons.md` §4). Juniors "accept agent output readily, sometimes without comprehension" ([arXiv 2602.00496](https://arxiv.org/pdf/2602.00496)).
- **Docs drift silently.** Agents "are prolific code changers but terrible at knowing what else their change affects" (Fiberplane drift, `scratchpad/notes/research-emergent-1.md` §1). Chattermax's README lists as "planned" features that shipped months ago (`notes/repo-chattermax.md`).
- **Confidence is decoupled from freshness.** mahdi answers authoritatively from 7-week-frozen state with unresolved merge conflicts its own index claims are fixed (`notes/repo-mahdi.md`). A senior treats that as a lead; a junior treats it as truth.

**The stake for this release:** the junior is the person with the *least* capacity to absorb undischarged cognitive debt, operating the *most* debt-generating kind of system (agent-heavy, multi-app). If the platform doesn't discharge the debt structurally, it lands on them — as incidents they can't diagnose, approvals they can't evaluate, and a system they're afraid to touch.

---

## 2. What the evidence says

### 2.1 Cognitive load theory applied to platforms (Team Topologies)

Team Topologies (Skelton & Pais) imported Sweller's cognitive load theory into org/platform design ([IT Revolution — Team Cognitive Load](https://itrevolution.com/articles/cognitive-load/), [DevOps Institute](https://www.devopsinstitute.com/team-cognitive-load/), [wind4change summary](https://wind4change.com/team-topologies-matthew-skelton-conway-law-cognitive-load-theory/)):

| Load type | Definition | Ops example for the junior | Platform's job |
|---|---|---|---|
| **Intrinsic** | Essential difficulty of the task itself | How HTTP, Postgres, k8s pods behave | *Minimize* via training, docs, sensible abstraction |
| **Extraneous** | Load imposed by how work is presented/delivered | Bespoke deploy steps per app, hunting for the right log, remembering which flag combination is safe | **Eliminate** — this is the platform's core job |
| **Germane** | Effort that builds durable knowledge/mental models | Understanding *why* app X falls over under load; incident close-out reflection | *Protect and maximize* |

The framework's prescription: platforms (as "Thinnest Viable Platform") exist to strip extraneous load so germane load can take the larger share. Platform engineering's concrete instrument is the **golden path / paved road**: an opinionated, supported, self-service route to "build/run something," pioneered at Spotify with Backstage — every golden path ships with a step-by-step tutorial and best-practice docs ([InfoQ on Spotify's paved paths](https://www.infoq.com/news/2021/03/spotify-paved-paths), [Jellyfish guide](https://jellyfish.co/library/platform-engineering/golden-paths/), [Mia-Platform on paved roads/guardrails](https://mia-platform.eu/blog/paved-roads-golden-paths-guardrails-railroads/)). Golden paths reduce cognitive load precisely by making the safe route the *default* route — repeatability replaces per-app memorization.

**Applied to this release:** the junior's extraneous load is anything that differs pointlessly across the 4–5 apps (deploy procedure, log location, metric names, runbook format, escalation route). The garden's job is uniformity: one card shape per app, one runbook format, one deploy verb (kploy), one approval surface. Their germane load — actually learning how these systems behave — is what mechanisms like evidence-rich approval cards and close-out notes must *feed*, not bypass.

A second Team Topologies insight matters here: **cognitive load is a capacity constraint, not a virtue signal.** One junior + agents is effectively a stream-aligned "team" of one; every garden component they must understand (Kafka? SurrealDB? feature-flagged builds?) is a direct draw against a fixed budget. The Bit Complete note independently sets the same bar: the garden's own runtime must be "at most one notch harder to run than kploy" (`research/notes/bitcomplete.md`; synthesis §3).

### 2.2 Cognitive/comprehension debt in agent-heavy development — the external evidence

- **Comprehension debt is now a named, studied phenomenon** ([Osmani](https://addyosmani.com/blog/comprehension-debt/), [O'Reilly](https://www.oreilly.com/radar/comprehension-debt-the-hidden-cost-of-ai-generated-code/), [arXiv 2604.13277 "Comprehension Debt in GenAI-Assisted SE Projects"](https://arxiv.org/pdf/2604.13277), [Allstacks](https://www.allstacks.com/blog/comprehension-debt-the-hidden-cost-of-ai-generated-code)). Key distinction from technical debt: it resides in cognition, not artifacts; it's invisible until demand exceeds understanding; and it accrues whenever understanding lags code growth.
- **The triple-debt model** ([arXiv 2603.22106](https://arxiv.org/pdf/2603.22106)) adds **intent debt**: when rationale isn't externalized, both humans *and future agents* lack the constraints/goals context. This is the strongest academic argument for the garden's decision-log-with-provenance practice — the wolfgang KB's "record why, not just what" convention is exactly intent-debt prevention.
- **ThoughtWorks put "complacency with AI-generated code" in the Hold ring**, citing GitClear (duplication and churn up, refactoring down) and Microsoft Research (AI confidence erodes critical thinking) (`notes/research-practitioner-lessons.md` §2).
- **The perception gap compounds it**: METR's RCT — experienced devs 19% *slower* with AI while believing they were 20% faster (`notes/research-practitioner-lessons.md` §2). Nobody's self-report of "I understand this system" is reliable; comprehension must be checked structurally (quizzes are impractical, but close-out notes and "explain before approve" are their operational cousins).
- **Mitigations the sources converge on** (Osmani; DX; LeadDev's "two kinds of debt"): document *why* not just *what*; at least one human must fully understand each change before deploy; treat verification as a structural constraint; use AI for conceptual inquiry, not just generation; watch the warning signs (hesitation to change things, black-box systems, tribal knowledge).

### 2.3 The garden's own evidence (discovery findings)

Every abstract failure mode above already has a concrete garden instance — this is not a hypothetical risk:

| Debt mechanism | Garden instance | Source |
|---|---|---|
| Stored state presented as truth, decoupled from age | mahdi: statuses stamped 2026-06-09/10, answered with "authoritative, only doubt yourself if accused of hallucinating" posture; 72 dirty git entries; index claims merge conflicts resolved that `git status` refutes | `notes/repo-mahdi.md` |
| Doc drift | Chattermax README lists shipped features (TLS, XEP-0198) as "planned"; docs reference ADRs not in the repo | `notes/repo-chattermax.md` |
| Context scatter / push-protocol atrophy | mahdi's update-log has one entry since January; the push-based update protocol "exists but is under-exercised"; sync happens via manual audits | `notes/repo-mahdi.md` |
| Undocumented release intent | The July release, junior persona, and 4–5-apps scenario appear **nowhere** in the wolfgang KB (verified by grep) — the system of record can't reason about its own release | `notes/kb-wolfgang.md` §6 |
| Snapshot docs as pseudo-state | mahdi's EXPLORATION-INDEX / IMPORT-SUMMARY point-in-time reports partially contradict newer statuses | `notes/repo-mahdi.md` |
| Configuration burden as extraneous load | Chattermax feature-flagged builds (`kafka`/`grpc`/`s3`) mean binary capabilities depend on compile flags — "an operational footgun"; trellis ships a 6-layer security model with sandbox **off** by default | `notes/repo-chattermax.md`, `notes/repo-trellis.md` |

And the garden also contains the **positive** prototypes:

- **Trellis's troubleshooting doc**: diagnostic command blocks + **copy-paste Claude Code prompts** — one runbook artifact serving human and agent (`notes/repo-trellis.md` §"Operations story"). Same pattern as Anthropic's internal plain-text-runbooks-plus-agent practice and the agents.md convention studied across 2,500+ repos (`notes/research-practitioner-lessons.md` §2–3), and as Flow-of-Action's SOP-grounded RCA agents (`notes/research-arxiv-multiagent-sdlc.md` §6).
- **Chattermax's CONTEXT_RESOLUTION.md**: 6 failure modes as symptom → diagnosis commands → causes → fixes — the strongest per-app runbook template in the garden (`notes/repo-chattermax.md`).
- **mahdi's per-project card**: `status.md` (with a mandatory **skip-level overview**) + `update-log.md` + `spec/` + `archive/` — a proven, junior-legible progressive-disclosure shape (`notes/repo-mahdi.md`).
- **wolfgang's decision log**: P0/D1–D8 with author, date, rationale, and supersession (D8 supersedes handoff-as-primitive) — provenance-first decision recording already in practice (`notes/kb-wolfgang.md` §4).
- **Greenwood's event-sourced design**: the lineage log is a "flight recorder" — the structural answer to "what did the agents do and why," which practitioners (Honeycomb Agent Timeline) are retrofitting painfully (`notes/research-practitioner-lessons.md` §3; `notes/kb-wolfgang.md` §3).

### 2.4 Knowledge freshness and authority calibration

The staleness deep-dive (`scratchpad/notes/research-emergent-1.md`) supplies the load-bearing mechanism set, all deterministic and public:

- **Two-date frontmatter** (`updated` vs `verified`): the gap between them is the primary machine-readable staleness signal (Google's SWE-book freshness dates; OKF + Curtner extension; Atlan).
- **Age-stamp every answer**: render provenance (entries used, verified dates, drift status) in the answer itself.
- **Derive, don't quote**: anything recomputable (git status, deploy versions, pod health) is recomputed at answer time; the KB stores only non-derivable knowledge (decisions, rationale) plus derivation pointers. This single rule would have prevented the mahdi incident outright.
- **Deterministic staleness gates with tiered behavior**: fresh → answer with stamp; past TTL → warning banner; badly stale on critical topics → refuse to answer authoritatively, emit the verification command instead.
- **Don't ask the LLM to track freshness**: deterministic freshness resolution beats LLM adjudication by +10.8 points like-for-like (FC-SH; the +28 figure is vs HippoRAG-v2 on multi-hop only — precision-corrected 2026-07-29) ([arXiv 2606.01435](https://arxiv.org/abs/2606.01435)); model-internal confidence misses stale-context failures.
- **Supersede, never overwrite, decisions** (Graphiti bi-temporal model): current vs historical is a metadata query, not an LLM judgment.

### 2.5 The oversight surface as a cognitive-load problem

Approval queues are where extraneous load concentrates. The evidence:

- **Approval fatigue is a documented security bug** — keep the junior's residual queue to a handful per day via DENY → ALLOW → human-residual routing (`notes/research-emergent-2.md`; synthesis §4.11).
- **Merge-Readiness Packs** (SASE, arXiv:2509.06216): every human gate receives a bundle — what/why/evidence/tests/rollback — so the adjudicator "can decide without reconstructing context" (`notes/research-arxiv-multiagent-sdlc.md` §2.3). This converts an open-ended comprehension task (high extraneous load) into a structured evaluation task (germane load).
- **The NeuBird on-call pattern**: the responder arrives to "a document outlining the explanation of what's happening and either giving me a solution or telling me who should get involved" (`notes/research-practitioner-lessons.md` §3) — pre-digested context replaces incident-time archaeology.
- **Explanations double as mentorship**: juniors lose the senior-explains-things channel; evidence-rich agent explanations partially replace it, *if* the junior engages actively — hence required close-out notes ("what happened / why," one line, feeds the KB) so the loop teaches rather than de-skills (`notes/research-practitioner-lessons.md` §6.8; arXiv 2602.00496 "review-as-learning").

### 2.6 Defaults over configuration

- Golden-path doctrine: the opinionated supported path *is* the cognitive-load reduction; configuration variety is extraneous load ([Jellyfish](https://jellyfish.co/library/platform-engineering/golden-paths/), [Mia-Platform](https://mia-platform.eu/blog/paved-roads-golden-paths-guardrails-railroads/)).
- The garden's counter-examples are its own components: trellis (6 security layers built, defaults off; `permission_mode: bypassPermissions`) and chattermax (capability-by-compile-flag). Codex's two-axis sandbox×approval model is the strongest shipped safe-default precedent (`notes/research-emergent-2.md` via synthesis §5.3).
- Kasava's expired docs-TLS-cert is a reminder that even competent small teams drop invisible operational balls — exactly the class of thing defaults + patrols should catch, not operator memory (`notes/competitor-kasava.md` via synthesis §2).

---

## 3. Recommendations: mechanisms the garden should ship

Organized by which load they attack. Each mechanism is cited to its evidence base above.

### 3.1 Kill extraneous load: uniformity and golden paths

1. **One per-app card, identical across all 4–5 apps** (mahdi's shape: skip-level overview → status → update-log → spec → archive). The junior learns the shape once; the fifth app costs nothing extra. This *is* progressive disclosure: overview first, drill-down on demand, archive out of sight (`notes/repo-mahdi.md`).
2. **One runbook format, dual-audience**: per-app runbooks in the trellis/chattermax style — symptom → diagnostic command block → causes → fixes, plus a copy-paste agent prompt per scenario. One artifact serves the human, the agent, and (per Flow-of-Action) the agent's SOP grounding (`notes/repo-trellis.md`, `notes/repo-chattermax.md`, arXiv survey §6).
3. **One verb per operation across apps**: deploy = kploy (canary + auto-rollback), metrics = Prometheus on a standard port with standard names (chattermax precedent), approvals = one surface delivered via tools the junior already uses (Slack/pin/bithub) (`research/notes/bitcomplete.md` via synthesis §3; `notes/repo-chattermax.md` §7).
4. **Defaults over configuration, safety-side**: sandbox on, `bypassPermissions` gone, budgets preset, one blessed build (no feature-flag capability matrix) for anything near the 4–5 apps. Every configuration decision the platform makes is one the junior doesn't carry (`notes/repo-trellis.md`; golden-path sources).
5. **Boring primitives for agent coordination**: git worktrees, branches, PRs, tests (CAID) — infrastructure the junior already has a mental model for; don't spend their load budget on bespoke coordination concepts (`notes/research-arxiv-multiagent-sdlc.md` §1.4).

### 3.2 Discharge comprehension debt: knowledge with provenance

6. **Two-date frontmatter (`updated` + `verified`, plus `owner`, `ttl_days`, `sources:` globs) on every KB entry** in wolfgang/mahdi — additive, OKF-aligned, shippable in days (`research-emergent-1.md` impl. 1).
7. **Age-stamped answers, unconditionally**: every navigator answer renders what it drew on, its verified dates, and drift status; warning banner past TTL; refusal + verification-command on badly-stale critical topics. Retire/patch the "only doubt yourself when accused of hallucinating" doctrine before any navigator faces the junior (`repo-mahdi.md`; `research-emergent-1.md` impl. 2 & 4).
8. **Derive-don't-quote**: runtime facts (deploy versions, git state, pod health) come from live commands at answer time, never from stored index text (`research-emergent-1.md` impl. 3).
9. **Decisions are append-only with provenance and supersession** — wolfgang already does this (P0/D1–D8 pattern); make it the garden-wide convention and the designated cure for intent debt ([arXiv 2603.22106](https://arxiv.org/pdf/2603.22106); `kb-wolfgang.md` §4).
10. **Chat-to-KB capture**: agent sessions and incidents end with a structured write-back (update-log entry or close-out note). The lesson of mahdi's unused push protocol: capture must be *forced by the workflow* (a gate won't close without the note), not left to discipline (`repo-mahdi.md`; Simply Business's KB-feeding fallback pattern, `research-practitioner-lessons.md` §2).

### 3.3 Protect germane load: make the system teach

11. **Evidence-rich approval cards (Merge-Readiness Pack shape)**: what/why/evidence (logs, metric deltas, test results)/blast radius/rollback plan — never "agent recommends X — approve?" (arXiv survey §2.3, §5.1).
12. **The NeuBird incident pattern as the on-call UX**: on alert, the junior opens a pre-built explanation document with recommended action and explicit escalation target (`research-practitioner-lessons.md` §6.2).
13. **Required one-line close-out on incident/approval completion**, feeding the KB — the active-engagement step the skill-formation literature says separates learning from atrophy (`research-practitioner-lessons.md` §6.8; Anthropic skill-formation study).
14. **Keep the residual human queue small** (DENY → ALLOW → human-residual; high initial patrol confidence thresholds) so the approvals the junior *does* see get genuine attention — germane load requires slack (`research-emergent-2.md`; LogicMonitor trust-threshold pattern).

### 3.4 Bound intrinsic load: don't ship concepts the operator must learn

15. **The garden's own runtime must not exceed the "one notch harder than kploy" bar.** Chattermax Phase 8 (Kafka+S3+gRPC, feature flags) is the named anti-pattern: "far too much operational surface" for this operator. Greenwood's D6 backing-store choice carries this tension — default to the embedded/simple tier; heavy pipeline strictly optional (`repo-chattermax.md` §5; `bitcomplete.md`; synthesis §5.5).
16. **Single-agent by default**: multi-agent topologies multiply both cost (~15×) and the operator's model of "what is running right now"; fan out only for read-only breadth-first investigation (`research-arxiv-multiagent-sdlc.md` §1.2, §8.1).
17. **Resume/rollback as one button.** "This agent died — resume it" is the junior's most common task; fold-the-log restore must surface as a UI/CLI affordance, with lifecycle state persisted by default (chattermax's memory-only freeze state is the anti-pattern) (`research-arxiv-multiagent-sdlc.md` §7; `repo-chattermax.md` §4).

---

## 4. What this means for the late-July release

Ordering principle: the mechanisms that are (a) cheapest to ship, (b) aimed at the junior's *first-week* failure modes (trusting stale answers, drowning in unexplained approvals, per-app snowflake procedures) come first. Everything here is consistent with — and mostly identical to — the synthesis's "pull to MVP" list (`00-research-summary.md` §6), viewed through the cognitive-load lens.

### Must-have (in the July release)

1. **Staleness-honest navigators** — two-date frontmatter on every KB entry; age/provenance stamp rendered in every answer; warning banner past TTL; derive-don't-quote for anything recomputable. *Days of work; fixes the single most dangerous junior-facing failure already observed (mahdi).* Includes amending the Navigator Authority doctrine.
2. **The uniform per-app card for all 4–5 apps** — skip-level overview + status + update-log + runbook link, same shape everywhere. This is the release's core progressive-disclosure artifact and the junior's home screen.
3. **One dual-audience runbook per app** — trellis/chattermax format: symptom → diagnostic commands → fix, plus copy-paste agent prompts. Even 3–5 scenarios per app (won't start, bad deploy, high error rate, cost spike, disk/cert) covers most of week one. Cert-expiry and backup checks included — the Kasava lesson.
4. **Evidence-rich approval cards** — every gated action ships what/why/evidence/rollback. Never a bare "approve?". (This is simultaneously the anti-over-trust countermeasure from the junior literature and the extraneous-load killer from SASE.)
5. **Safe defaults flipped** — sandbox on, no `bypassPermissions`, preset per-task/per-day budget caps, one blessed build per component near the apps. Zero new configuration surface exposed to the junior.
6. **Required close-out notes** — one line ("what happened / why") on incident close and high-risk approvals, written into the app's update-log. Enforced by the workflow (the gate doesn't close without it), not by discipline.
7. **Write the release itself into the wolfgang KB** — the July scope, junior persona, and 4–5-apps scenario are currently absent from the system of record; the planning navigator can't discharge intent debt about a release it doesn't know exists (`kb-wolfgang.md` §6).
8. **Resume as a button** — agent lifecycle state persisted by default; "resume" a first-class CLI/UI affordance.

### Deferrable (post-July hardening)

- **CI-time doc-drift detection** (Fiberplane-style AST anchors, Dosu freshness scoring, frontmatter validators failing builds) — the warning-banner tier covers July; mechanical drift-checking is the durable backstop, ship it in August/September.
- **Bi-temporal decision graphs** (Graphiti-style validity intervals) beyond the simple `superseded_by` frontmatter convention.
- **Per-message governance / lineage UI (Greenwood M3)** — the differentiator, but not required for the junior's cognitive load in July; the event log substrate (M1) plus the mechanisms above carry the release.
- **Autonomy promotion machinery** (ITIL standard-change register; track-record-based trust tiers) — start everything at "agent explains, junior decides"; earn autonomy later.
- **AI-review layer compressing the approval queue** — valuable at scale; at 4–5 apps with high patrol thresholds the raw queue should already be small.
- **Comprehension checks / structured learning loops** beyond close-out notes (e.g., periodic "explain this system back" reviews with Terra) — good practice, not platform work.
- **Full Backstage-style portal** — the per-app card set + Slack/pin/bithub delivery is the thin viable version; a portal is post-productization.

### The one-line test for every release decision

*Does this reduce what the junior must hold in their head, or add to it?* Every component, default, and document in the July release should pass it. The garden's differentiator framing (safe-by-default agent ops for non-expert operators) and its cognitive-debt framing are the same thesis: the platform absorbs the extraneous load, records the intent, and keeps the human's remaining effort germane — understanding the apps, not fighting the tooling.

---

## Sources

### Garden notes (paths absolute)
- `/Users/terra/Developer/wolfgang/research/software-garden-2026-07/00-research-summary.md` — synthesis
- `/Users/terra/Developer/wolfgang/research/software-garden-2026-07/notes/repo-mahdi.md` — staleness/authority failure; per-app card; skip-level overview
- `/Users/terra/Developer/wolfgang/research/software-garden-2026-07/notes/repo-trellis.md` — runbook-with-agent-prompts pattern; inverted safety defaults
- `/Users/terra/Developer/wolfgang/research/software-garden-2026-07/notes/repo-chattermax.md` — doc drift; runbook template; ops-surface anti-pattern
- `/Users/terra/Developer/wolfgang/research/software-garden-2026-07/notes/kb-wolfgang.md` — decision-log provenance; release absent from KB
- `/Users/terra/Developer/wolfgang/research/software-garden-2026-07/notes/research-arxiv-multiagent-sdlc.md` — Merge-Readiness Packs; junior over-trust; CAID; review bottleneck
- `/Users/terra/Developer/wolfgang/research/software-garden-2026-07/notes/research-practitioner-lessons.md` — NeuBird pattern; METR perception gap; skill atrophy; ThoughtWorks Hold
- `/Users/terra/Developer/wolfgang/research/software-garden-2026-07/notes/research-emergent-1.md` — freshness mechanisms (two-date, derive-don't-quote, staleness gates)
- `/Users/terra/Developer/wolfgang/research/software-garden-2026-07/notes/research-emergent-2.md` — approval fatigue; risk tiers (via synthesis)
- `/Users/terra/Developer/wolfgang/research/software-garden-2026-07/notes/bitcomplete.md` — "one notch harder than kploy" operability bar (via synthesis)
- `/Users/terra/Developer/wolfgang/research/software-garden-2026-07/notes/competitor-kasava.md` — expired-TLS-cert lesson (via synthesis)

### Web (fetched or searched 2026-07-29)
- Osmani, *Comprehension Debt* — https://addyosmani.com/blog/comprehension-debt/ (also [O'Reilly Radar](https://www.oreilly.com/radar/comprehension-debt-the-hidden-cost-of-ai-generated-code/))
- DX, *Cognitive debt: the hidden risk in AI-driven software development* — https://getdx.com/blog/cognitive-debt-the-hidden-risk-in-ai-driven-software-development/
- *From Technical Debt to Cognitive and Intent Debt* — https://arxiv.org/pdf/2603.22106
- *Comprehension Debt in GenAI-Assisted SE Projects* — https://arxiv.org/pdf/2604.13277
- Kosmyna et al., *Your Brain on ChatGPT: Accumulation of Cognitive Debt* — https://arxiv.org/abs/2506.08872 / https://www.brainonllm.com/
- IT Revolution, *Team Cognitive Load* — https://itrevolution.com/articles/cognitive-load/
- DevOps Institute, *Understanding and Reducing Team Cognitive Load* — https://www.devopsinstitute.com/team-cognitive-load/
- Wind4Change, *Team Topologies & cognitive load theory* — https://wind4change.com/team-topologies-matthew-skelton-conway-law-cognitive-load-theory/
- InfoQ, *Spotify's paved paths* — https://www.infoq.com/news/2021/03/spotify-paved-paths
- Jellyfish, *How to build golden paths developers will actually use* — https://jellyfish.co/library/platform-engineering/golden-paths/
- Mia-Platform, *Paved roads, golden paths, guardrails and railroads* — https://mia-platform.eu/blog/paved-roads-golden-paths-guardrails-railroads/
- *Don't Ask the LLM to Track Freshness* — https://arxiv.org/abs/2606.01435 (via research-emergent-1)
- *From Junior to Senior: Allocating Agency* — https://arxiv.org/pdf/2602.00496 (via arXiv survey note)
