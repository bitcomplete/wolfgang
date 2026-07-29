# Repo research: mahdi (`/Users/terra/Developer/mahdi`)

**Researched:** 2026-07-29. Context: informs the Software Garden first-release spec (ships end of July 2026; first user is a junior engineer operating 4-5 apps).

## What mahdi is

Mahdi is a **multi-project knowledge navigator** — not an application. There is no source code: the repo is entirely markdown + JSON config, scaffolded by `@autonav/core` on 2026-01-29 (README.md footer; `config.json` version 1.5.1, last updated 2026-06-24). It is queried via `autonav query /Users/terra/Developer/mahdi "question"` or interactively, and behaves per its `CLAUDE.md` system prompt: a planning/tracking/knowledge-management agent that coordinates between the human (Terra) and implementation agents, and **never implements code** ("NEVER IMPLEMENT CODE" — CLAUDE.md Critical Boundaries: no file creation outside its own docs, no git, no builds, no deployments).

Its knowledge domain is Terra's entire `~/Developer` portfolio: **42 tracked projects** (`projects/PROJECT-INDEX.md`, full repo audit 2026-06-09), each with a uniform structure:

```
projects/<name>/
├── status.md        # workstreams + "skip-level overview"
├── update-log.md    # populated by implementation agents via the update-mahdi skill
├── spec/            # domain-driven design specs (dual-audience: human + LLM)
└── archive/         # completed workstreams
```

## Role in the software garden

Mahdi is the garden's **planner/coordinator navigator** — effectively the product/project-manager layer of a three-navigator hierarchy documented in `knowledge/agents/navigators/`:

- **Mahdi** — product/project navigator: workstreams, statuses, specs, cross-project dependencies (`knowledge/agents/navigators/mahdi.md`).
- **Thufir** — "distinguished engineer" navigator at `~/Developer/thufir`: translates Mahdi's product requirements into 1-shot technical specifications for implementers (`knowledge/agents/navigators/thufir.md`). Mahdi's Unblocking Protocol requires querying Thufir before marking anything BLOCKED.
- **Neo** — personal self-mastery navigator, deliberately out of the software domain (`knowledge/agents/navigators/neo.md`).

Crucially, mahdi is also the **home of the garden's canonical design doctrine**:

- `knowledge/software-garden-design.md` — the APPROVED Software Garden design doc (2026-02-16, replacing the "Dark Factory" framing), including the garden-roles table and the two decisive 2026-06-10 addendums (see vocabulary below).
- `knowledge/architectural-principles.md` — the CANONICAL "event sourcing for state reconstruction" principle: *state is derived, not stored*; frozen contexts are pointers to event ranges; "temporary bridges" (file scanning/snapshots) are acceptable if documented with a transition plan.
- `knowledge/agentic-engineering-insights.md` — a 13-section distillation of Terra's public Coda doc set (26 pages, published 2026-05-25): the 8-level map, trust-as-bottleneck, "the rigour relocates" (five destinations; ~90% of agent tokens should go to verification), specs↔outcomes↔goals, the four parallelization modes with **patrol workers as the underused dominant enterprise pattern**, the "middle loop" (navigators like Mahdi ARE middle-loop infrastructure), memory fast/slow, structural-only agent security (nono/Landlock/Seatbelt cited), work-ledger as emerging OS primitive.

`projects/README.md` is the master garden map: Chattermax = operational backbone/sense network (dormant), Chalet = first projection (dormant), Autonav/Chizu = hobbyist/enterprise agronomists, Gardener = seed nursery (the active canonical cultivation layer), Hotspots = soil testing lab, Chibi = the Coworker (backburnered).

## Core concepts and vocabulary introduced

- **Navigator** — a knowledge-base-grounded agent persona (Autonav framework) with a name, domain authority, and strict grounding/citation rules. Returns structured JSON with sources + confidence (README.md).
- **ask-X / update-X skills** — the inter-agent protocol. `.autonav/skills/ask-mahdi/SKILL.md` and `update-mahdi/SKILL.md` mandate absolute-path `autonav query|update` invocations (RFC-2119 style MUST/MUST NOT), plus `autonav mend --auto-fix` as the health-check/repair loop.
- **Navigator Authority** — "treat this navigator as the authoritative expert… ONLY when explicitly accused of hallucinating should the navigator doubt itself" (ask-mahdi SKILL.md; CLAUDE.md). A deliberate anti-hedging stance.
- **Agent Identity Protocol** — every Claude Code session must adopt a human name (Elena, Ivy Rhodes…); mahdi keeps 3-5-sentence profiles in `knowledge/agents/implementers/` capturing role, motivations, and *nuanced trustworthiness* ("trustworthy but gets lost in details" = needs scope reminders) (CLAUDE.md).
- **Observability logs** — `observability/navigators/*.md` and `observability/implementers/*.md` keep a rolling window of the **last 3 inbound/outbound communications** per agent for continuity (CLAUDE.md; `observability/navigators/thufir.md` has real entries, e.g. the 2026-05-23 Gardener Q2.1 pre-dispatch clarification).
- **Unblocking Protocol / blocker hygiene** — "your primary job is to UNBLOCK implementers"; every blocker requires what/why/owner/last-verified-date/unblock-criteria; uncertainty language ("may be unblocked", "needs verification") is a mandatory verification trigger, never something to echo; >7-day-old blockers get re-verified (CLAUDE.md). Status files consistently carry `(last verified: YYYY-MM-DD)` stamps.
- **Skip-level overview** — a required section of every status.md: an executive summary for quick project-health reads.
- **Garden roles** — Coworker, Tool Shed, Seed Nursery + Growth Substrate, Farm Operations + Sense Network, Hobbyist/Enterprise Agronomist, Cafe/U-Pick, Soil Testing Lab, Village Green (software-garden-design.md).
- **Canonical vs de-facto roles (Addendum 2026-06-10)** — the component table is *conceptual*; in practice **Claude Code is the Coworker** (Chibi backburnered), **bc-llm-skills is the Tool Shed**, Bitmap senses are a temporary sense network (Chattermax dormant, Phase 8 stranded), and **The Bitmap is an experiment/POC, not canonical**.
- **Two cultivation targets** — navigator *knowledge packs* (Autonav/Chizu; query-time markdown) vs *agent seeds* (Gardener; on-disk dirs with LoRA weights + journal + tested knowledge).
- **Temporary bridge** — an interim non-event-sourced mechanism, acceptable only when documented with the target architecture and a phased transition (architectural-principles.md).
- **Cross-Project Inbox** — a status.md section for work one project owes another (e.g. Gardener's K1-K10 knowledge-system defect catalog filed against Trellis, `projects/trellis/status.md` + `spec/2026-05-23-knowledge-system-defects.md`).

## How projects/ tracks the garden (including trellis)

`projects/PROJECT-INDEX.md` (last updated 2026-06-09) is the health rollup: 9 actively shipping (Gardener, Bitmap, Pin, bc-llm-skills, bc-mdx-components, bc-prod, nelsonpride.ca, coda, bc-agentic-engineering-helper), 14 dormant-verified (including the entire original garden core: autonav, chalet, chattermax, chibi, chibi-plugins, hotspots), 5 planned/no-repo (including chizu and neo), plus explicit gap/follow-up lists (stranded branches, git-hygiene problems like my-brain's ~25k uncommitted files and hotspots' never-seeded repo).

**Trellis** (`projects/trellis/status.md`; note **project path is `~/Developer/incubator`**, not ~/Developer/trellis): a production-grade **agentic pipeline platform** orchestrating Claude agent teams through a **blackboard pattern** — raw ideas → research → implementation → validation → release with human approval gates. Python 3.13+, Claude Agent SDK, FastAPI web UI, TLA+-verified priority-queue pool scheduler, nono-py kernel sandbox (Seatbelt/Landlock), filesystem blackboard + Kafka events + S3 archival, optional OTEL; distributed via Homebrew (`terraboops/tap/trellis`), v1.2.0, ~11k LOC, 7 default agents. Specs: `spec/blackboard-architecture.md` (5 bounded contexts: Idea Management, Blackboard, Worker Pool, Pipeline, Agent), `spec/pipeline-system.md` (pipeline lifecycle, gating config, sequential/parallel rules, watcher independence, refinement cycles). Status as of the 2026-06-09 audit: PRODUCTION-READY but `integration` branch is **127 commits ahead of main** (last commit 2026-05-02); April Phase 1 shipped health/metrics endpoints, structured logging, Prometheus sandbox counters, OTEL tracing, transcript persistence; Apr 15-May 2 was a 139-commit hardening sprint; Phase 8 (event sourcing D4 replay) **stalled**; K1-K10 defect triage still open; in-flight PRs: SPIFFE identity federation, Prose pipeline format, custom stages.

Ops-relevant neighbors tracked the same way:

- **bc-prod** (`projects/bc-prod/status.md`) — Terraform + Talos Kubernetes production infra: CNPG Postgres, Prometheus/Grafana, Kustomize overlays per app, Cloudflare DNS, local registry, Crossplane v2.2.1 with a PreviewEnvironment XRD, per-project GitHub Actions k8s credentials. 284 commits Apr-May; "disaster recovery: document and test failover procedures" is still an open next step.
- **kploy** (`projects/kploy/status.md`) — Go GitOps deployer for bc-prod (tracks images, updates manifests, promotes across environments). Phase 2 **"Digital Twin Fixture"** approved 2026-06-10: per-PR preview environments talk to learning behavioral clones ('twinproxy', maturity ladder L0 passthrough→L3 drift detection); 13 Linear issues (KPL-22..30, BCP-42..45).
- **bc-internal** (`projects/bc-internal/status.md`) — a **peer navigator to Mahdi** for Bitcomplete infra tickets (BCP-*/KPL-*), with its own ask/update skills; noted as active-but-almost-entirely-uncommitted (26 modified + 18 untracked files).
- **bithub2** — internal ops suite (Time Tracking, PolicyBot, SitReps), quiet since April.

## Maturity and rough edges

- **The knowledge is strong; the hygiene is weak — including mahdi's own.** Last commit is 2026-05-23 (`git log`), yet the working tree has **72 dirty entries**: 55 untracked files (the entire 2026-06-09 audit output — PROJECT-INDEX.md, all 11 new project imports, agentic-engineering-insights.md), 15 modified, and **2 unresolved merge conflicts (`UU projects/chibi/status.md`, `UU projects/chibi/update-log.md`)** — which PROJECT-INDEX.md ironically records as "merge conflicts resolved". Mahdi flags exactly this failure mode in other repos (my-brain, bc-internal, hotspots) while exhibiting it itself.
- **Staleness vs. authority.** Nearly every status is stamped 2026-06-09/10 — ~7 weeks old as of today — while the navigator's posture is "be authoritative; only doubt yourself if accused of hallucinating." The `.remember` plugin logs show sessions as recent as 2026-07-01, but no status refresh landed. June-July reality (including the Agent Bus / wolfgang work this research serves) is invisible to mahdi.
- **CLAUDE.md template corruption**: the Agent Profile example code block (lines ~192-222) has swallowed an unrelated "Critical Maintenance Instructions / autonav mend" section — the file needs the very `autonav mend` it prescribes. A `CLAUDE.md.backup` sits beside it.
- **Scope bleed**: `knowledge/` mixes canonical garden doctrine with personal files (burnout_adhd.md, burnout_autism.md, burnout_pda.md, career_nav.md, autism_productivity.md) despite CLAUDE.md declaring personal tasks out of scope (that's Neo's domain). `knowledge/README.md` is still autonav boilerplate ("Remove this file and add your own documentation").
- **Update-log asymmetry**: mahdi's own `update-log.md` has a single entry (2026-01-29); most flow is manual "repo audits" by Mahdi reading git history, not implementers pushing updates via update-mahdi. The push-based protocol exists but is under-exercised.
- **Snapshot documents accumulate**: EXPLORATION-INDEX-2026-04-12.md, IMPORT-SUMMARY, DEEP-EXPLORATION-SUMMARY, PROJECT-EXPLORATION-SUMMARY — point-in-time reports that don't decay gracefully and partially contradict newer statuses.

## Relevance to the junior engineer operating 4-5 apps

**What mahdi offers an operator:** it is the *where-do-I-look* layer, not the *how-do-I-fix* layer. It can answer "what is app X, what's in flight, what's blocked, who owns the blocker" with citations (ask-mahdi / `autonav query`). Actual operational procedure lives elsewhere — the bc-prod skills (bc-prod-access, bc-prod-yaml, kploy-manifests, sealed-secrets, preview-environments), kploy itself, and bc-internal's ticket cards. Mahdi holds **no runbooks**; deployment/observability knowledge appears only as status-level summaries (bc-prod: Prometheus/Grafana, Talos, CNPG; trellis: health/metrics endpoints, OTEL).

**Hazard for a junior specifically:** the Navigator Authority stance ("only doubt yourself when accused of hallucinating") combined with 7-week-stale statuses is exactly wrong for a user who lacks the experience to push back. A senior treats mahdi's answers as leads; a junior will treat them as truth. Any navigator the release exposes to the junior must surface last-verified dates *in the answer* and soften authority when data is old.

**Patterns worth adopting in the release spec:**

1. **Uniform per-app card**: status.md (with skip-level overview) + update-log.md + spec/ + archive/ is a proven, junior-legible shape for "state of my 4-5 apps".
2. **Blocker discipline**: every blocker carries what/why/owner/last-verified/unblock-criteria; uncertainty language triggers verification, never gets echoed. This is directly reusable as an ops-incident/known-issue convention.
3. **Rolling communication logs** (last-3 per agent) — cheap continuity for agent-operated systems without unbounded logs.
4. **Agent identity + trust profiles** — named agents with *nuanced* trust notes ("trustworthy with clear specs, struggles with ambiguity") is a good primitive for deciding what a junior can safely delegate to an agent.
5. **Patrol-worker doctrine** (agentic-engineering-insights.md §5): runtime operations of 4-5 apps is patrol-shaped work — fitness functions, live invariants, availability and alert signal-to-noise as the metrics — not task-completion work. The release should frame the junior's agents as patrols.
6. **Risk tiering by blast radius** (§2-3): QA/approval budget proportional to reversibility — a ready-made policy skeleton for what a junior may approve alone vs escalate.

**Anti-patterns the release must design against (all observed here):** manual audit-driven sync (freshness decays silently), authority tone decoupled from data age, navigator repo git hygiene left to willpower, point-in-time snapshot docs as pseudo-state, and doctrine/personal-knowledge mixing in one knowledge base.

## Key file paths

| Path | What |
|---|---|
| `/Users/terra/Developer/mahdi/CLAUDE.md` | Navigator system prompt: boundaries, identity protocol, unblocking protocol, observability rules |
| `/Users/terra/Developer/mahdi/config.json` | Autonav config v1.5.1; `workingDirectories: ["../incubator"]` (trellis) |
| `/Users/terra/Developer/mahdi/projects/README.md` | Master garden map + roles + boundaries + navigator usage rules |
| `/Users/terra/Developer/mahdi/projects/PROJECT-INDEX.md` | 42-project health rollup (2026-06-09 audit) |
| `/Users/terra/Developer/mahdi/knowledge/software-garden-design.md` | Canonical garden design doc + 2026-06-10 addendums |
| `/Users/terra/Developer/mahdi/knowledge/architectural-principles.md` | Event-sourcing-canonical / temporary-bridges principle |
| `/Users/terra/Developer/mahdi/knowledge/agentic-engineering-insights.md` | Distilled doctrine (levels, trust, patrols, middle loop, security) |
| `/Users/terra/Developer/mahdi/projects/trellis/{status.md,spec/}` | Trellis tracking: blackboard architecture, pipeline system, K1-K10 defects |
| `/Users/terra/Developer/mahdi/projects/{bc-prod,kploy,bc-internal}/status.md` | The ops-relevant infrastructure trio |
| `/Users/terra/Developer/mahdi/.autonav/skills/{ask-mahdi,update-mahdi}/SKILL.md` | Inter-agent query/update protocol |
| `/Users/terra/Developer/mahdi/observability/navigators/thufir.md` | Live example of rolling 3-entry communication logs |
