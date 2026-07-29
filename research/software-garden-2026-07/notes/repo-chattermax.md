# Repo research: chattermax

Repo: `/Users/terra/Developer/chattermax` (github.com/terraboops/chattermax, Apache 2.0, author Terra Tauri)
Read: 2026-07-29. Last commit: `4a26c31` on 2026-03-09 ("feat(events): Phase 8 D2 - S3 archival pipeline") — the repo has been dormant for ~4.5 months.

## What it is

Chattermax is a **modern XMPP server written in Rust** (RFC 6120/6121, targeting Conversations/Gajim-class clients) whose distinguishing feature is a **hook system for AI agent integration**. The same binary also has a CLI client mode for scriptable one-shot XMPP operations (`send`, `presence`, `join`, `leave`, `roster`, `listen`). Sources: `README.md`, `docs/architecture.md`, `chattermax-client/src/main.rs`.

Three-crate Cargo workspace, all at v0.1.0:

- `chattermax-core` — shared XMPP types: JIDs, stanzas, stream state, stream management (`sm.rs`), plus **`types/` (the 12 structured agent message types)** and **`chizu/` (context-resolution client)**.
- `chattermax-server` — the server: async Tokio, SQLite or PostgreSQL persistence, SASL PLAIN auth, router with in-memory session registry (`HashMap<bare_jid, Vec<Session>>`), MUC (XEP-0045), MAM (XEP-0313), disco (XEP-0030), offline queue, TLS (`tls/`), Prometheus metrics (`metrics.rs`), hooks (`hooks/`), freeze/thaw (`freeze/`, `thaw/`), context resolver, event store (`event_store/`), gRPC query API (`grpc/`), S3 archival (`archival/`).
- `chattermax-client` — CLI client + client library (used by the Chibi plugin for outbound sends).

## Role in the software garden

Chattermax is the **communication substrate / message bus of the garden's earlier iteration**: agents are XMPP users (JIDs), collaborate in MUC rooms, and exchange *typed* structured messages. It is explicitly designed as the middle of a three-part system:

- **Chibi** = execution engine (LLM agent runtime). Integration doc: `docs/CHIBI_INTEGRATION.md`. Chibi sends via a `chattermax_send` tool; inbound XMPP messages are delivered into Chibi's unified `inbox.jsonl`. Plugin lives at `~/Developer/chibi-plugins/chattermax/`.
- **Chizu** = knowledge server. Chattermax resolves `chizu://` context URIs over HTTP and injects knowledge packs into spawned agent processes (`docs/CONTEXT_RESOLUTION.md`).
- **Chattermax** = the bus: routing, hooks, archives, presence, and agent lifecycle (freeze/thaw).

This is exactly the triad that inspires the Agent Bus design in wolfgang: transport (chattermax), execution (chibi), knowledge (chizu).

## Core concepts and vocabulary

### Hooks (`docs/hooks.md`, `chattermax-server/src/hooks/`)
- Server-side event → filter → **spawn subprocess** → JSON on stdin, JSON reply on stdout.
- Configured in `chattermax.toml` (`[[hooks]]` with `name`, `command`, `timeout` default 30 s, `filters` (regex/glob on `type`/`from`/`to`/`body`/`room`), `env`, `working_dir`).
- Response actions: `reply`, `send` (to arbitrary JID/room), `none`, or an `actions` array.
- Design principles stated in `docs/architecture.md`: process isolation, language agnostic, simple protocol, timeout protection, filter-before-spawn.

### The 12 structured message types (`chattermax-core/src/types/message.rs`, `docs/CHIBI_INTEGRATION.md`)
`thought`, `tool_call`, `tool_result`, `todo`, `code_change`, `integration`, `review_comment`, `work_available`, `question`, `answer`, `status_update`, `feature_complete` — each with a typed payload struct (e.g. `TodoPriority`, `ReviewSeverity`, `CodeChangeType`). Plus lifecycle messages `FreezeNotification` / `ThawRequest`. Messages carry `metadata` with an optional **`correlation_id`** (request/response linking) and an optional **`context_ref`** (`chizu://` URI). On the wire they ride as custom XML payloads inside XMPP messages (e.g. `<question xmlns="jabber:x:chibi:question">`).

### Context resolution (`docs/CONTEXT_RESOLUTION.md`, `chattermax-core/src/chizu/`, `chattermax-server/src/context_resolver.rs`)
- `chizu://context-id[/section][@version]` URI scheme; maps to HTTP GETs against `CHIZU_BASE_URL` returning a `KnowledgePack` JSON (name, version, description, map of file-path → content).
- `ContextResolver` with **hybrid LRU + TTL cache** (defaults: 100 entries, 5-minute TTL; env `CHATTERMAX_CONTEXT_CACHE_SIZE` / `_TTL`).
- Resolved context is written to a temp file and handed to spawned Chibi processes via `CHIBI_CONTEXT_PATH`.
- Stated philosophy: **graceful degradation — "Messages are more important than context"**; Chizu outages never block delivery, agents must detect a missing context and adapt.

### Freeze/Thaw (agent lifecycle) (`docs/FREEZE_THAW.md`, `src/freeze/`, `src/thaw/`)
- An agent "freezes" by sending a `FreezeNotification` (reason enum: TaskComplete | Error | UserRequest | Timeout | Unknown; conversation context: room, participants, last message id; `active_context_ref`). `FreezeHandler` stores `FrozenAgentState` **in an in-memory HashMap** and returns a `freeze_id` (UUID).
- A `ThawRequest` (freeze_id, target agent JID, optional resurrection room, requestor, `additional_context` instructions) makes the `ResurrectionService` write a context JSON to `/tmp/chibi-context-*.json`, set env vars (`CHIBI_CONTEXT_PATH`, `CHATTERMAX_FREEZE_ID`, `CHATTERMAX_AGENT_JID`, `CHATTERMAX_RESURRECTION_ROOM_JID`, `CHATTERMAX_ACTIVE_CONTEXT_REF`), and **spawn a Chibi process** that resumes with the frozen state.
- Frozen state survives re-thaw (stays in map) but **not a server restart** — persistence is listed as a known gap.

### Event pipeline (Phase 8, the newest work — last 4 commits)
- `EventStore` trait (`src/event_store/traits.rs`) mirroring the `DatabaseBackend` pattern; events: `message_sent`, `groupchat_message`, `room_joined`, `room_left`, `presence_update`, `message_queued`. Router publishes events.
- Backends: `InMemoryEventStore` (default/testing) and `KafkaEventStore` (rdkafka, idempotent producer, `acks=all`) behind `--features kafka`.
- **gRPC Event Query API** behind `--features grpc` (default port 50051).
- **S3 archival pipeline** behind `--features s3` (implies kafka): consumer-group batching (1000 events / 60 s), zstd compression, MinIO/LocalStack-compatible `endpoint_url`. Config in `chattermax.toml` (all commented out by default).

## Maturity assessment

**Docs say "MVP (v0.1.x)".** More precisely:

- **Solid / done**: core 1:1 + MUC messaging, MAM, disco, offline queue, SQLite backend, PostgreSQL backend (merged PR #11, `docs/POSTGRESQL.md` — connection pooling, migration guide), hooks, Prometheus metrics, stream management (XEP-0198 — `docs/STREAM_MANAGEMENT.md`, `sm.rs`, with DB-persisted `stream_sessions`), TLS module with certificate-expiry metrics, CI (check/fmt/clippy `-D warnings`/unit + integration tests/release build/e2e via go-sendxmpp — `.github/workflows/ci.yml`), integration test suites for context, freeze/thaw, and stream management.
- **README is stale relative to code**: it lists TLS and XEP-0198 as "Planned/In Progress" though both are implemented; it doesn't mention freeze/thaw, context resolution, events/Kafka/gRPC/S3, or metrics at all. The docs/ folder (UPPERCASE files) is where the real design lives.
- **Rough edges**: `test_output.txt` and `test_serialization_debug.rs` committed at repo root (debug cruft); a local `chattermax.db` in the working tree (untracked); frozen-agent state is memory-only; MUC state is memory-based; simple password hashing (Argon2/bcrypt listed as future); single-server only, no federation; per-message subprocess spawn for hooks (no pooling — listed as a Phase 4 enhancement); `/tmp` context-file accumulation is a documented operational hazard; docs reference ADRs (ADR-0003/0004/0008) that are **not in this repo**.
- Feature-flagged builds (`kafka`, `grpc`, `s3`) mean the shipped binary's capabilities depend on how it was compiled — an operational footgun.

## Operations relevance (junior engineer running 4-5 apps)

What chattermax already provides that maps to the release's operator story:

- **Deployment**: `docs/deployment.md` is a genuinely runbook-grade guide — systemd unit with hardening (`NoNewPrivileges`, `ProtectSystem=strict`), Docker/Compose recipes, nginx stream-block TLS termination, UFW/firewalld rules, DNS SRV records. Resource floor is tiny (256 MB RAM min).
- **Observability**: Prometheus exporter on **port 9090 by default** (`config.rs::default_metrics_port`); metric families `xmpp_connections_total`, `xmpp_active_sessions`, `xmpp_auth_attempts_total`, `xmpp_stanzas_processed_total`, `xmpp_errors_total`, `xmpp_messages_routed_total`, TLS cert expiry/validity gauges, and `archival_*` counters. Logs via `tracing` → journalctl.
- **Runbooks**: the troubleshooting sections in `docs/deployment.md` and especially `docs/CONTEXT_RESOLUTION.md` (6 diagnosed failure modes with symptom → diagnosis commands → causes → fixes) are the strongest runbook prototypes in the garden so far — a good template for what the release should ship per-app.
- **Backup**: SQLite online-backup one-liner + retention script; PostgreSQL guide for production.
- **How agents participate in operations**: the hook system is the mechanism — an ops agent is just a hook filtered on a room (e.g. `ops` → `operations@conference.local` in the Chibi plugin's `room_mappings`), receiving events as JSON and replying with actions. Freeze/thaw gives operators a pause/inspect/resume lever over agents, and the resurrection-room concept lets a human summon a frozen agent into a debug room with extra instructions.

## Implications for the July release / Agent Bus design

1. **Vocabulary worth carrying forward**: typed message taxonomy (12 types + lifecycle), `correlation_id`, `context_ref`, `work_available` (work-stealing primitive), freeze/thaw + resurrection, inbox as a unified append-only format, "messages > context" graceful degradation.
2. **The hook model is the strongest, simplest agent-integration primitive**: process isolation, language agnostic, JSON stdin/stdout, filter-before-spawn, timeouts. It is easy for a junior engineer to test manually (`echo '{"type":"message","body":"@bot hi"}' | /path/to/hook`).
3. **XMPP itself is heavy baggage**: much of the codebase is XEP compliance for Android chat clients (carbons, entity caps, stream management) — irrelevant to an agent bus. The Chibi integration already tunnels custom XML payloads through XMPP awkwardly; docs/CHIBI_INTEGRATION.md's own "Future Enhancements" ask for msgpack/protobuf and delivery guarantees. Agent Bus should keep the semantics and drop the protocol.
4. **Persistence gaps are exactly the kind that burn a junior operator**: in-memory freeze state (lost on restart, and docs' troubleshooting explicitly names "server restarted → frozen states lost"), memory-based MUC state, `/tmp` context-file accumulation. The release must persist agent lifecycle state by default.
5. **The Phase 8 event pipeline (EventStore trait → Kafka → S3 with gRPC query) shows the intended provenance/audit direction**, but its feature-flag + Kafka + S3 dependency stack is far too much operational surface for a junior engineer running 4-5 apps — the Agent Bus equivalent should default to an embedded store (the InMemory/SQLite tier) with the heavy pipeline strictly optional.
6. **Doc drift is a real cost**: README vs. code divergence in a ~5-month-old repo shows why wolfgang's knowledge-base-first approach matters; the release should treat the navigator KB as the source of truth for what is actually deployed.
7. **Prometheus-by-default worked here**: metrics on 9090 with named, described metrics and cert-expiry gauges is a pattern to standardize across the 4-5 apps.

## Key file index

| Topic | Path |
|---|---|
| Overview | `/Users/terra/Developer/chattermax/README.md` |
| Architecture | `/Users/terra/Developer/chattermax/docs/architecture.md` |
| Hooks | `/Users/terra/Developer/chattermax/docs/hooks.md`, `chattermax-server/src/hooks/` |
| Chibi integration + 12 message types | `/Users/terra/Developer/chattermax/docs/CHIBI_INTEGRATION.md`, `chattermax-core/src/types/message.rs` |
| Context resolution (chizu://) | `/Users/terra/Developer/chattermax/docs/CONTEXT_RESOLUTION.md`, `chattermax-core/src/chizu/` |
| Freeze/Thaw | `/Users/terra/Developer/chattermax/docs/FREEZE_THAW.md`, `chattermax-server/src/{freeze,thaw}/` |
| Deployment/runbooks | `/Users/terra/Developer/chattermax/docs/deployment.md` |
| PostgreSQL | `/Users/terra/Developer/chattermax/docs/POSTGRESQL.md` |
| Stream management | `/Users/terra/Developer/chattermax/docs/STREAM_MANAGEMENT.md` |
| Metrics | `/Users/terra/Developer/chattermax/chattermax-server/src/metrics.rs` |
| Event pipeline | `/Users/terra/Developer/chattermax/chattermax-server/src/{event_store,grpc,archival}/` |
| Config reference | `/Users/terra/Developer/chattermax/chattermax.toml`, `docs/configuration.md` |
