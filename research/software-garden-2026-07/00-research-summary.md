# Software Garden Research Summary — Synthesis for the Late-July 2026 Release

**Date:** 2026-07-29
**Purpose:** Synthesize all discovery, competitive, and literature research into one report informing the software-garden release spec. Frame: a first usable iteration ships end of July 2026; the first user is a **junior engineer running runtime operations of 4–5 apps** on Bit Complete's platform.
**Sources:** the 13 notes files cited inline throughout (paths in §8).

---

## 1. What the software garden is, and its current state

"The software garden" carries **two meanings** in the corpus, and the ambiguity itself is a finding:

1. **The legacy prototype family** — Chattermax, Chalet, Chizu, Gardener, Bitmap, Hotspots — Bit Complete's earlier agent-infrastructure experiments, deliberately backburnered 2026-06-10 and now mined as prior art (`notes/kb-wolfgang.md` §2, `notes/repo-mahdi.md`).
2. **The new ecosystem vision** — the thing shipping in July: tools for growing and operating software with heavy agent involvement. Critically, **this vision is written down nowhere in the wolfgang KB**: no July date, no junior-operator persona, no 4–5-apps scenario appears anywhere in it (`notes/kb-wolfgang.md` §6, verified by grep). The release goal exists only in this research thread and must be written into the KB.

### Component state (as of 2026-07-29)

| Component | What it is | State | Source |
|---|---|---|---|
| **mahdi** | Multi-project knowledge navigator and home of canonical garden doctrine (design doc, architectural principles, agentic-engineering insights, 42-project index) | Knowledge-rich but **7 weeks stale** (statuses stamped 2026-06-09/10), 72 dirty git entries including 2 unresolved merge conflicts its own index claims are fixed; authoritative tone decoupled from data age | `notes/repo-mahdi.md` |
| **trellis** (repo dir `incubator`) | Agentic pipeline platform: ideas → agent teams → released software, blackboard pattern, TLA+-verified scheduler, 6-layer security model, health endpoints, cost dashboard, v1.3.1 on Homebrew | The most mature and ops-ready garden component — but 127 unpushed commits on `integration`, single-user/no-RBAC, and **sandbox off by default with `bypassPermissions` agents** | `notes/repo-trellis.md` |
| **chattermax** | Rust XMPP server with AI-agent hooks: 12 typed message types, freeze/thaw lifecycle, `chizu://` context resolution, Kafka/S3 event pipeline (Phase 8) | Dormant ~4.5 months; MVP-grade; strongest runbook prototypes in the garden, but persistence gaps (memory-only freeze state) and a stale README | `notes/repo-chattermax.md` |
| **wolfgang / Greenwood** | Planning navigator whose sole design work is **Greenwood**, the new Agent Bus: event-sourced agent runtime + coordination bus on Kafka (Annals log, Rootlines lineage DAG, Grieve governance, Coldframe evals, Graft adapters) | Design largely settled (P0 confirmed; D7/D8 confirmed; D1–D6 proposed pending Terra); ~40 implementation-ready tickets in a draft work breakdown; **nothing built**; Linear untouched pending authorization | `notes/kb-wolfgang.md` |

**The through-line:** the garden's *ideas* are strong and internally consistent (event sourcing, typed messages, lineage, human gates, patrol agents), but its *hygiene* is weak — stale statuses, unpushed branches, doc drift, dormant repos. The release inherits both.

## 2. Competitive landscape

Headline: **no direct competitor exists for the release's actual job** — a junior engineer operating running apps with agent help. All three researched "competitors" are adjacent, not overlapping.

- **"Hive" is actually Buzz (Block, Jack Dorsey)** — launched 2026-07-21. Nostr-based chat + Git + agents-as-first-class-channel-members workspace. It is a *collaboration* substrate, not an ops tool: no deploys, monitoring, incidents, or app lifecycle. Pre-1.0 (v0.4.2x, 174 open issues in 3 days), no fine-grained tool authorization, no approval gates. **Worth borrowing:** per-agent cryptographic identity with signed per-action audit trails; the model-agnostic ACP harness bridging Claude Code/Codex/Goose. **Watch:** its `buzz-workflow`/approval-gate roadmap. (`scratchpad/notes/competitor-hive.md`)
- **Kasava (kasava.dev)** — tiny SF startup (2–10 people); "PRDs from your real code." Agentic planning for PMs/product engineers; explicitly "AI is an exoskeleton, not a coworker." Zero runtime-ops surface. Cautionary lessons: broad "agentic platform" positioning narrowed to one concrete job within a year; 522-point HN blog posts produced no visible product traction; their docs domain had an **expired TLS cert** — precisely the failure class the garden's ops tooling should catch. (`notes/competitor-kasava.md`)
- **Efecto (efecto.app)** — Pablo Stanley's solo, free, agent-first design canvas. Philosophically closest to the garden ("agents *are* the canvas" — domain state as first-class agent-addressable objects; watch-then-intervene live legibility; worker agents + a reviewer agent), but a creative tool with no governance story at all. Its zero-governance posture is fine when blast radius is an ugly artboard; fatal for prod ops — which is exactly the garden's differentiation space. (`research/software-garden-2026-07/notes/competitor-efecto.md`)

**Differentiation thesis (supported across all three notes + `notes/research-emergent-2.md` §10):** the garden's defensible position is **safe-by-default agent operations for non-expert operators** — risk-tiered approval gates, least-privilege brokered credentials, lineage-backed audit, cost caps. Buzz and Efecto punt on governance entirely; Kasava doesn't touch ops; and academic protocol analysis (arXiv 2606.31498) documents real governance gaps in MCP/A2A/ACP — audit/replay, human escalation, deliberation/dissent — so bus-level governance is differentiation, not NIH. *(Corrected 2026-07-29: that paper does NOT cover budget/delegation expressiveness as round 1 claimed — see `notes/research-round2-stat-verification.md` item 11. The budget/delegation gap still holds via the orchestration survey and industry reports; cite those for it, not 2606.31498.)*

## 3. Bit Complete's position

(`research/notes/bitcomplete.md`)

- Small (best estimate low-teens headcount), bootstrapped, profitable Toronto consultancy (founded 2020, Dylan Trotter + Matt Schweitz, ex-Google/YouTube/Slack). Founders are systems/runtime people — a Kafka-based agent runtime is in their wheelhouse.
- It already runs an internal platform the release must sit on: **bc-prod** (shared k8s), **kploy** (GitOps deployer), **pin**, **bithub**, Autonav navigators. Two open SRE roles signal ops is a real, growing cost center — the exact economic slot the "junior + agents operate 4–5 apps" release targets.
- The garden is **pre-announcement internal IP** with zero public footprint. The July date is a self-imposed internal milestone: scope can trade against the date without external cost, but the first user's trust cannot.
- Consequences: (a) the release is best read as an *internal-productivity milestone with productization optionality* (the kploy/Labs trajectory), judged by hours saved on real client-app ops, not launch optics; (b) with no platform team, the garden's own runtime must be **at most one notch harder to run than kploy**, or it consumes the SRE capacity it's meant to free; (c) surface agent activity through pin/bithub/Slack/kploy — interfaces the junior already touches — rather than new UIs.

## 4. Strongest findings from the research

### Academic (`notes/research-arxiv-multiagent-sdlc.md`)

1. **Greenwood's theoretical footing is verified.** *From Spark to Fire* (arXiv:2603.04474) exists, says what the KB claims, and its genealogy-graph message-layer governance prevents final infection in **≥89% of runs** — direct validation of lineage-as-first-class-data and D8 (governance at the message layer).
2. **Multi-agent is conditional, not default.** MAST (arXiv:2503.13657, 1,600+ annotated traces): MAS gains over single agents are "often minimal"; failures are system-design failures. Anthropic: multi-agent works mainly by spending more tokens, at ~15× cost. The 2026 winning pattern (icat-agent) is a cheap rubric deciding single-agent vs. team — matching D7's bus-side decomposition.
3. **Coordinate via boring SE primitives.** CAID (arXiv:2603.21489): isolated git worktrees + branch-and-merge + test-verified integration beat bespoke shared state (+25.6% PaperBench). Plans as reviewable artifacts; SASE's "Merge-Readiness Packs" (what/why/evidence/tests/rollback) as the shape of every human gate.
4. **Verification: grounded, early, placed — not maximal.** Delayed/ungrounded correction *destabilizes* consensus into oscillation (arXiv:2606.27409); verifiers must be evidence-anchored (tests/logs/metrics) and placed at topological hotspots the bus can compute. Reframes "who verifies the verifier" as dose-and-placement tuning over the event log.
5. **Ops agents diagnose worse than they mitigate.** SREGym: frontier agents at 38.9–72.6% diagnosis success on realistic SRE faults — the quantitative argument for "agent proposes, gate disposes."
6. **Juniors mis-calibrate trust in both directions.** arXiv:2602.00496 (N=10, qualitative; corrected 2026-07-29 from the earlier "over-trust" reading): novices struggle between over-reliance *and* overly cautious avoidance. Anthropic's skill-formation study: passive AI use impairs debugging skill. Design consequence: the interface must make confidence legible — preventing rubber-stamping *and* freezing — not merely add friction.

### Practitioner (`notes/research-practitioner-lessons.md`)

7. **Instructions are not guardrails.** The Replit prod-DB deletion happened during an explicit prompt-level freeze; enforcement must be structural (credentials, environment separation, gates in the execution path). Corroborated by Amazon Q, Clinejection, GitLab Duo: everything an agent reads — including app logs during investigation — is untrusted input.
8. **The field landed exactly where the release should:** 696-practitioner survey — only 8% run operational AI agents in prod; 62% want co-pilot not autopilot; what works is investigation/triage agents producing explanation documents, with humans keeping judgment calls (the NeuBird on-call pattern).
9. **Measure, don't trust self-reports.** METR RCT: developers 19% slower with AI while believing they were 20% faster.

### Emergent deep-dives

10. **Staleness must be metadata-conditioned, not model-judged** (`scratchpad/notes/research-emergent-1.md`): two-date frontmatter (`updated` vs `verified`), age-stamped answers, derive-don't-quote anything recomputable, tiered staleness gates. "Don't Ask the LLM to Track Freshness" (arXiv:2606.01435): deterministic freshness resolution beats LLM adjudication by +10.8 points like-for-like (FC-SH, gpt-4o-mini; the oft-quoted +28 is specifically vs HippoRAG-v2 on the multi-hop variant — precision-corrected 2026-07-29). This directly fixes the observed mahdi failure mode.
11. **Risk tiering has a citable public consensus** (`notes/research-emergent-2.md`): OWASP's 4-level action taxonomy + approval-bound-to-exact-action record schema; CSA's 6 autonomy levels (release ships at L2 for mutating work, L3 for read-only patrols); ITIL's standard-change register as *the* principled autonomy-promotion mechanism (autonomy earned per action-type on evidence, granted by humans); Codex's sandbox-mode × approval-policy two-axis model as the strongest shipped default. Approval fatigue is a documented security bug — keep the junior's queue to a handful/day via DENY → ALLOW → human-residual routing.
12. **Every documented runaway-cost incident is the absence of a cap plus invoice-time discovery** (`notes/research-emergent-3.md`): $14k/day leaked-key incident, $6.5k autonomous-provisioning incident (InfoQ-verified). Budget enforcement must fail closed at the gateway (unpriceable calls are a bypass class); detection at action time, never bill time; Anthropic's own baselines (~$13/dev/active-day, 90% of users <$30/day) are day-one "is this normal?" anchors for the dashboard.

## 5. Tensions and contradictions between sources

1. **Milestone order vs. user need (the big one).** Wolfgang's work breakdown puts everything the operator needs — dashboards, crash detection, rate limits, cost budgets, sandboxing — in M3/M6, *last* (`notes/kb-wolfgang.md` §6). Every other strand (SRE survey, incident case studies, cost incidents, junior-trust findings, durability practice) says those are MVP-critical *first* features. The arXiv survey states it flatly: "that ordering is backwards for a July ship with this user."
2. **Navigator authority vs. staleness.** Mahdi's doctrine says "be authoritative; only doubt yourself when accused of hallucinating" while answering from 7-week-frozen state (`notes/repo-mahdi.md`). The freshness research says authority must be computed from verification metadata and degraded when stale (`research-emergent-1.md`). A senior treats stale confidence as a lead; the junior will treat it as truth. Doctrine must change before the release exposes a navigator to the junior.
3. **Trellis's defaults vs. its own security design.** Trellis *ships* a six-layer defense-in-depth model, then defaults to `sandbox_enabled: false` and `permission_mode: bypassPermissions` (`notes/repo-trellis.md`). Public best practice (Codex) defaults the other way. The parts exist; the defaults are inverted (`research-emergent-2.md` §5).
4. **Greenwood's differentiator vs. the July date.** Per-message governance — the point of the design, and the thing the protocol-gap literature validates — is M3, three milestones deep. A July "first usable iteration" is realistically M1 (single-agent, event-sourced, resumable), which ships the substrate but not the differentiator (`notes/kb-wolfgang.md`).
5. **Kafka-based bus vs. small-company operability.** Greenwood assumes Kafka + projections + (open) graph-DB choices; Chattermax's Phase 8 already showed a Kafka+S3+gRPC stack is "far too much operational surface" for this operator (`notes/repo-chattermax.md` §5); the Bit Complete note sets the bar at "one notch harder than kploy." The backing-store decision (D6, still proposed) carries this tension.
6. **Multi-agent bus vs. multi-agent skepticism.** The garden is building coordination infrastructure for many agents while MAST/Cognition find multi-agent gains often minimal and Anthropic prices them at ~15×. Reconciliation exists — Greenwood governs *whatever* topology runs, and the release should default to single-agent-per-task — but the spec should say so explicitly rather than assume fleets.
7. **Agents for a junior vs. agents de-skilling juniors.** The release bets on a junior + agents replacing senior attention; the literature (skill-formation RCT, "AI handicaps junior engineers," learning-debt sources) warns passive AI use erodes exactly the judgment the junior needs. The countermeasure set (evidence-rich approval cards, required close-out notes feeding the KB, explanations-as-mentorship) is design work the KB doesn't yet contain.
8. **"Software garden" naming collision.** The KB's "garden" = legacy prototypes; the release's "garden" = the new ecosystem. Until the KB defines the latter, every planning query resolves the term to the wrong referent (`notes/kb-wolfgang.md` §2).
9. **Minor factual frictions:** the "Hive" competitor brief was a misnomer (it's Buzz); Greenwood's $22k/yr governance figure is modeled-not-measured (externally consistent but unconfirmed — label it); several headline stats (89% cascade prevention, SREGym ranges) were read from abstracts and need PDF verification before external use. *(Verification completed 2026-07-29 — `notes/research-round2-stat-verification.md`: 89% and the SREGym range VERIFIED against full texts; CAID +25.6% VERIFIED; the 2606.31498 budget/delegation claim broke and has been corrected throughout; three secondary-only stats marked do-not-cite.)*

## 6. Implications for the late-July release (prioritized)

Frame: junior engineer, 4–5 apps on bc-prod/kploy, ~low-teens-person company, no external announcement pressure.

### What appears DECIDED in the KB (safe to build on)

- **P0:** event sourcing end-to-end; state is only ever a fold of the log (Terra, 2026-06-30).
- **D7:** bus-side claim decomposition, mechanical provenance, classifier triage (Terra, 2026-07-02).
- **D8:** messaging as the only inter-agent primitive; governance per-message (Terra, 2026-07-02).
- **Kafka API** as the substrate; three-parties boundary (agent / verifier / dumb bus); caching not a design driver.
- **Milestone ordering M1→M6** exists as a draft critical path (but see OPEN — the ordering itself is challenged by this research).
- **Process guardrails:** no Linear ticket creation without Terra's authorization; work breakdown is the ticket source.

### What is OPEN (needs Terra / needs writing into the KB)

1. **The release definition itself.** July scope, the junior-operator persona, and the 4–5-apps scenario are absent from wolfgang. Write them in as decisions/open questions first — otherwise the planning navigator cannot reason about its own release.
2. **Which milestones are in.** Recommendation from the evidence: ship **M1-level Greenwood scope** (event spine, runtime, resume) and do *not* pretend M3 governance arrives by July — but pull forward a minimal ops layer (items 3–6 below) that doesn't depend on Greenwood at all.
3. **Risk-tiered action gates — pull to MVP.** Adopt the 4-tier matrix as versioned policy-as-code at the tool-call boundary: T1 read/diagnose auto (patrols live here), T2 reversible writes auto+log+rate-limit, T3 junior approves from an evidence pack, T4 two-person rule (junior + Terra). Ship "agent explains, junior decides," never "agent fixes." Flip trellis defaults (`sandbox_enabled: true`, no `bypassPermissions`) for anything near the 4–5 apps. (`research-emergent-2.md`, practitioner + arXiv notes.)
4. **Cost caps — pull to MVP.** Hard per-task ($5–20 default) and per-agent daily caps, fail-closed, gateway-enforced, with a dashboard answering "is this normal?" against published baselines. Tighter budgets for overnight/unattended runs. (`research-emergent-3.md`.)
5. **Staleness-honest navigators — pull to MVP.** Two-date frontmatter on every KB entry, age/provenance stamps rendered in every answer, derive-don't-quote for anything recomputable (git status, deploy versions), warning-banner tier for stale entries. Shippable in days; fixes the single most dangerous junior-facing failure observed. (`research-emergent-1.md`.)
6. **Resume/rollback as an operator button.** Greenwood's fold-the-log restore is ahead of industry bolt-on checkpointing, but the junior's most common task is "this agent died — resume it"; make it a UI/CLI affordance, and persist agent lifecycle state by default (Chattermax's memory-only freeze state is the named anti-pattern).
7. **Two blocking technical decisions before any build:** runtime language/stack (blocks P1/P2) and D1–D6 confirmations. Secondary opens: lineage store, claim-state retention (before M2), verifier trust model, real message-rate measurement, S3-path produce latency.
8. **Reuse, don't invent.** Per-app card (status/update-log/spec, skip-level overview) from mahdi; runbook style from trellis's troubleshooting doc and chattermax's deployment docs (diagnostic blocks + copy-paste agent prompts — one artifact serving human and agent); approvals delivered via Slack/pin/bithub; deploys only via kploy with canary+auto-rollback so reversibility is manufactured and deploys demote from T4 toward T2.
9. **Design against the junior's predicted failure modes:** evidence-rich approval cards (never "agent recommends X — approve?"), default-deny on irreversible ops, required one-line close-out notes feeding the KB (teach, don't de-skill), high initial patrol confidence thresholds (three bogus findings in week one kills trust), and a defined break-glass emergency path (or the junior bypasses the system during the first real incident).
10. **Positioning (when it matters):** lead with the concrete job ("run these 4–5 apps safely with agents"), not the ecosystem vision — the Kasava lesson. The governance-first wedge is uncontested by Buzz/Kasava/Efecto and validated by the protocol-gap literature; keep it the spine of the story.

## 7. One-paragraph verdict

The garden's design doctrine is unusually well-aligned with where the 2024–2026 research and practitioner consensus landed — lineage-based message-layer governance, risk tiering, patrol agents, event-sourced flight recorders are all now externally validated. The gap is not conceptual but *operational and sequential*: the KB schedules operator-facing safety last, the strongest existing component ships with its safety rails off, the knowledge layer answers confidently from stale state, and the release itself is undefined in the system of record. The late-July spec's job is therefore mostly re-sequencing and defaults-flipping: define the release in wolfgang, pull gates/caps/staleness-honesty/resume into MVP, ship Greenwood at M1 scope honestly, and let the differentiator (per-message governance) land after the junior already trusts the system.

## 8. Notes files cited

| File | Subject |
|---|---|
| `/Users/terra/Developer/wolfgang/research/software-garden-2026-07/notes/repo-mahdi.md` | mahdi navigator + garden doctrine |
| `/Users/terra/Developer/wolfgang/research/software-garden-2026-07/notes/repo-trellis.md` | Trellis pipeline platform |
| `/Users/terra/Developer/wolfgang/research/software-garden-2026-07/notes/repo-chattermax.md` | Chattermax XMPP agent bus prototype |
| `/Users/terra/Developer/wolfgang/research/software-garden-2026-07/notes/kb-wolfgang.md` | Wolfgang KB / Greenwood design state |
| `/Users/terra/Developer/wolfgang/research/software-garden-2026-07/notes/competitor-hive.md` | "Hive" → Buzz (Block) |
| `/Users/terra/Developer/wolfgang/research/software-garden-2026-07/notes/competitor-kasava.md` | Kasava.dev |
| `/Users/terra/Developer/wolfgang/research/software-garden-2026-07/notes/competitor-efecto.md` | Efecto.app |
| `/Users/terra/Developer/wolfgang/research/software-garden-2026-07/notes/bitcomplete.md` | Bit Complete company profile |
| `/Users/terra/Developer/wolfgang/research/software-garden-2026-07/notes/research-arxiv-multiagent-sdlc.md` | Academic survey, multi-agent SDLC |
| `/Users/terra/Developer/wolfgang/research/software-garden-2026-07/notes/research-practitioner-lessons.md` | Practitioner lessons, agents in delivery/ops |
| `/Users/terra/Developer/wolfgang/research/software-garden-2026-07/notes/research-emergent-1.md` | Knowledge freshness / authority calibration |
| `/Users/terra/Developer/wolfgang/research/software-garden-2026-07/notes/research-emergent-2.md` | Risk-tiered approval gates / least privilege |
| `/Users/terra/Developer/wolfgang/research/software-garden-2026-07/notes/research-emergent-3.md` | Cost governance for agent fleets |
