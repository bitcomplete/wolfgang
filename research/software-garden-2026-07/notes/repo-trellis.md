# Repo notes: trellis (at `/Users/terra/Developer/incubator`)

Research date: 2026-07-29. Read for the software-garden July release spec (first user: a junior engineer operating 4-5 apps).

## What it is

Trellis is an **agentic pipeline platform**: you submit an *idea* (a short title + description) and configured teams of Claude agents research, build, test, and release it autonomously, with human approval gates between phases. Tagline: "A structure for growing ideas with agent teams." (`README.md`)

- Python package (`pyproject.toml`: name `trellis`, **version 1.3.1**, Python >= 3.12, Apache-2.0) plus a **Tauri desktop app** shell (`src-tauri/`, `homebrew/Casks/trellis.rb`).
- The repo directory is called `incubator` because that was the project's former name — trellis ships a `trellis migrate-project` command that renames `.incubator` markers, `pool/incubator.log`, etc. to trellis names (`README.md` "Migrating from incubator", `docs/self-hosting.md`).
- Distributed via `brew install terraboops/tap/trellis` (Python CLI) and `brew install --cask trellis` (desktop app, `terratauri/tap`); public GitHub identity is `terraboops/trellis`.

## Core architecture (from `docs/architecture.md`, `docs/agents.md`)

**Design philosophy** (README): filesystem-first, no framework, blackboard pattern, human-in-the-loop, agents as plain text, TLA+-verified scheduling.

- **Blackboard pattern** — all state is files under `blackboard/ideas/<slug>/`: `status.json` (phase, pipeline config, cost, history), `idea.md`, `feedback.json`, phase artifacts (self-contained `.html`), `agent-logs/` (full transcripts), `agent-knowledge/<agent>/`. No database for coordination; a projection store (SurrealDB dep in `pyproject.toml`; `trellis/core/projection.py`) serves dashboard reads.
- **Agents are Claude sessions** run via `claude-agent-sdk` (`trellis/core/agent.py` `BaseAgent`). Each agent = `agents/<name>/prompt.py` (a plain `SYSTEM_PROMPT` string constant), `.claude/CLAUDE.md`, and a `knowledge/` dir. Defined centrally in `registry.yaml` (model, tools, `max_turns`, `max_budget_usd`, `permission_mode: bypassPermissions`, cadence, sandbox fields).
- **Pipelines** — ordered agent stages per idea, with `post_ready` watchers, `parallel_groups`, and per-stage **gating** (`auto` / `human-review` / `llm-decides`). Default pipeline: `ideation → implementation → validation → release`. Templates live in `pipeline-templates/` in **YAML or "Prose"** — Prose is a small declarative orchestration DSL (Lark parser at `trellis/core/prose_parser.py`) that compiles to the same dict.
- **Worker pool** (`trellis/orchestrator/pool.py`) — `POOL_SIZE` concurrent slots, priority scoring (phase weight, starvation, deadline pressure, iteration diminishing returns), state snapshot to `pool/state.json` every 10s. Scheduler correctness is formally specified in TLA+ (`specs/pool_scheduler.tla` + `.cfg` + `tla2tools.jar`).
- **Phase recommendations** — after each run an agent recommends `proceed` / `iterate` / `needs_review` / `kill`; 3 iterations force human review.
- **Watchers** — cron-cadence agents (`competitive-watcher` every 6h, `research-watcher` daily) that keep monitoring ideas *after release* and feed structured feedback back into pipeline agents; `artifact-check` is a `phase: "*"` cross-cutting quality agent.
- **Evolution** — `trellis evolve` runs retrospectives over agent transcripts and updates `agents/<name>/knowledge/learnings.md`, which is injected into future runs (self-curating knowledge loop).
- **Human-in-the-loop channels** — Telegram bot (`trellis/comms/telegram.py`) or web dashboard approval.

### Newer feature branch: custom stages + pluggable handlers (`FEATURES.md`)

A major generalization landed on `feat/custom-stages` (165 automated tests / 176 enumerated behaviors):

- Pipelines become `stages[] → role_groups[][] → roles`, each role bound to a **handler type**: `agent`, `script` (subprocess, JSON-in/JSON-out over stdin/stdout), `webhook` (HTTP POST), `human` (writes a pending request file, pauses), or `k8s_job` (Kubernetes Job with injected client adapter). This is the key move from "agent pipeline" toward a general orchestration substrate.
- Retry/timeout policy per role (exponential/linear/fixed backoff), atomic role execution via `BlackboardSnapshot` (copy-on-run + lock file + rollback; crash recovery scans `.trellis-locks/`), watcher `scope` rules and watcher-driven **reactivation from done**.
- Deferred non-goals recorded explicitly: secrets for handlers, RBAC (**trellis remains a single-user app**), per-blackboard git repos, real k8s wiring, full multi-stage editor UI.

## UI (`docs/screenshots/`)

Warm cream/off-white design, "Powered by Trellis" footer. Nav: **Ideas / Activity / Pool / Agents / Pipelines / Costs / Evolution / Settings** + "New Idea" button.

- `trellis-ideas.png` — idea cards with status badges (SUBMITTED / IMPLEMENTATION / READY), a 4-step progress rail (ideation → build → validate → release) with check/spinner icons, watcher chips (artifact check, competitive, research), and per-idea priority score, **cost in dollars**, and iteration count.
- `trellis-agents.png` — agent cards (name, description, model tier like "4-6", turn limit, cron cadence), "Create with Wizard" (LLM generates agent config from a description) and "Browse Plugins" buttons, plus an inline "1 registry migration available — Run `trellis migrate` or apply here" banner.

Stack: FastAPI + WebSocket backend, Jinja2 + HTMX + Tailwind frontend (`trellis/web/`). Routes include ideas, agents, pool, costs, evolution, pipelines, migrations, settings, decisions, health (`trellis/web/api/routes/`).

## Desktop app (`docs/tauri.md`, `src-tauri/`)

Tauri (Rust) shell that spawns `trellis serve` as a **sidecar**: picks a loopback port, mints a per-launch bearer token passed via `--auth-token-stdin`, waits for a `--emit-ready-line` JSON handshake, then navigates the webview; the web bootstrap sends the token on every fetch/WebSocket. Release pipeline builds signed/notarized `.app` bundles for a Homebrew cask (Developer-ID or ad-hoc signing modes). Phase 3 items (project picker, parent-death detection, universal binary) are deferred.

## Security model (`docs/security.md`) — notable and garden-relevant

Six defense-in-depth layers for autonomous agents:

1. **L1 nono sandbox** — kernel-level isolation (Seatbelt on macOS, Landlock on Linux); per-role filesystem/network/command restrictions, irreversible once applied.
2. **L2 credential proxy** — agents never see raw API keys; nono's localhost proxy injects auth headers upstream (`sandbox_proxy_credentials`, 1Password/Keychain credential maps).
3. **L3 SDK tool policy** (`trellis/core/tool_policy.py`) — `can_use_tool` allow/deny with a Bash blocklist (sudo, osascript, keychain dumps, `rm -rf /`, pkill...) and per-role directory scoping.
4. **L4 audit** — `pool/audit.jsonl` (append-only JSONL of SDK tool calls) + nono's own audit store for OS-level activity.
5. **L5 attack-surface reduction** — per-agent tool lists (ideation gets no Bash; watchers are read-only).
6. **L6 attestation** — agent instruction files (prompts, CLAUDE.md, registry.yaml) Sigstore-signed in CI; nono verifies signatures at runtime (`trust-policy.json`).

Sandbox is **off by default** (`sandbox_enabled: false`) to avoid breaking existing deployments.

## Operations story (`docs/self-hosting.md`, `docs/troubleshooting.md`, `AGENTS.md`)

- Run modes: `trellis serve` (dashboard + pool), `trellis serve --background` (daemon, PID at `pool/trellis.pid`, log at `pool/trellis.log`), `trellis run` (pool only), launchd/systemd units, nginx reverse-proxy recipe (`WEB_HOST=127.0.0.1`).
- **Health/observability**: `/healthz` (liveness), `/readyz` (readiness), `/metrics` (Prometheus text format — ideas by phase, total cost, sandbox failure count) in `trellis/web/api/routes/health.py`; optional OpenTelemetry tracing via the `otel` extra (`trellis/otel.py`, no-op when unset).
- **LLM-narrated logs** (`trellis/human_log.py`): log records are batched and sent to Claude, which emits natural-language narration; formats are named system-prompt specs, project-overridable in `log-formats/<name>.md`; `trellis serve --list-log-formats`.
- **Troubleshooting doc is runbook-shaped**: diagnostic command blocks (grep patterns on `pool/trellis.log`, `pool/state.json` inspection, `trellis status <idea>`), and — notably — **copy-paste prompts for Claude Code** covering general agent diagnostics, stuck-idea diagnosis, sandbox permission issues, and cost investigation. Agents are expected to participate in operating the system itself.
- Config via `.env` (`TELEGRAM_BOT_TOKEN`, `POOL_SIZE`, `JOB_TIMEOUT_MINUTES`, `MODEL_TIER_HIGH/LOW`).
- Release checklist in `AGENTS.md`: pytest, `/test-trellis` browser test, log-format listing, and health endpoints must respond before a version bump.

## Maturity and rough edges

- Actively developed and reasonably mature for a solo project: v1.3.1, real Homebrew distribution, TLA+ spec, big test suite (the custom-stages branch alone claims 165 test cases incl. Playwright browser flows), health endpoints, migration tooling for old projects and registries.
- **Working tree state (as read)**: branch `integration`, **127 commits ahead of origin/integration** (unpushed), last commit dated 2026-05-02, with local modifications to `src-tauri/Cargo.toml`, `trellis/core/agent.py`, `trellis/core/blackboard.py` and an untracked `pipeline-templates/default.yaml`. Recent commit log is dominated by defensive hardening ("coerce null/corrupt X so Y can't 500/crash") — signals the data-shape layer was brittle and is being toughened.
- Rough edges explicitly on record: single-user, no RBAC, no handler secrets management; sandbox off by default; k8s handler ships without a wired client; desktop app Phase 3 (project picker, parent-death handling, non-arm64 builds) deferred; placeholder app icons; cask sha256 is `REPLACE_ME`; pipeline editor UI only edits stage 0.
- Naming split-brain: repo dir `incubator`, GitHub `terraboops/trellis` in README vs `terratauri/incubator` in the cask/tauri docs.

## Role in the software garden

The wolfgang research README (`/Users/terra/Developer/wolfgang/research/README.md`) describes "the software garden" as a family of Bit Complete prototypes (Chattermax, Chalet, Chizu, Gardener) treated as prior art for the new Agent Bus/Greenwood design; trellis is not mentioned there by name, so its exact slot is my characterization: trellis is the garden's **growing/incubation organ** — it takes ideas from seed to released software with agent teams, then keeps them under continuous watcher surveillance post-release. Its vocabulary (blackboard, pipeline, stage/role/handler, gating, watcher, evolution, phase recommendation) and its patterns (filesystem-first state, plain-text agents, human gates, kernel sandboxing with credential proxying, LLM-narrated ops logs, agent-assisted runbooks) are the strongest existing embodiment of "growing software with agents" and a direct vocabulary/pattern source for the garden release.

## Implications for the July release + junior-engineer operator

1. **The troubleshooting doc is the model for garden runbooks**: short diagnostic command blocks plus copy-paste Claude Code prompts. A junior operator can paste a prompt and let an agent read `pool/state.json`, logs, and transcripts to diagnose. Replicate this pattern for every garden app.
2. **Ops surface is junior-friendly in shape**: one process per project, `serve --background/--stop`, `/healthz`, `/readyz`, `/metrics` ready for a shared Prometheus/Grafana; systemd/launchd/nginx recipes are already written. But it is **single-user with no auth story beyond the desktop-app bearer token** — putting a dashboard on a shared host needs a reverse proxy plus network-level protection.
3. **Cost visibility is first-class** (per-idea `$` on the idea list, Costs page, `cost_usd` per transcript, budget caps per agent) — exactly what a junior engineer needs to notice a runaway agent; the cost-investigation Claude prompt is a ready-made runbook.
4. **Failure containment exists but must be switched on**: sandbox default-off, `bypassPermissions` agents. If trellis-style agents run anywhere near the 4-5 production apps, the release spec should mandate `sandbox_enabled: true` + credential proxy per agent, since a junior operator can't be expected to judge blast radius.
5. **Stuck-state handling is explicit and inspectable**: iteration cap → human review, failed-role Retry button + `trellis retry <idea> <role>`, crash recovery via lock rollback. Good precedent: every stuck condition has both a UI affordance and a CLI counterpart.
6. **The handler contract (script/webhook/human/k8s_job, JSON-in/JSON-out)** is the natural bridge between trellis and the rest of the garden — external systems (deploys, checks, Agent Bus messages) can be pipeline roles without being Claude agents.
7. **Watch the unpushed state**: 127 unpushed commits on `integration` and a dirty tree mean "what's installed via brew" and "what's in this repo" can differ substantially; the release spec should pin which trellis version ships.
