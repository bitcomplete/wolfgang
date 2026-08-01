---
doc_type: knowledge
topic: Substrate — event sourcing on a swappable log; Postgres first, Kafka scrapped
tags: [event-sourcing, p0, logstore, postgres, redis, kafka, envelope, schema, d9]
created: 2026-08-01
updated: 2026-08-01
status: current — DECIDED (P0 2026-06-30; D9 2026-07-29); implementation is R1 work (GP1)
summary: P0 stands - state is only ever a fold of the log. D9 - the log backend is swappable (LogStore trait); Postgres first (zero new deployables); Redis a candidate; Kafka scrapped ("probably never need it"). Envelope reserves identity/signature fields day one.
---

# Substrate: event sourcing on a swappable log

## P0 — event sourcing end-to-end (DECIDED 2026-06-30, unchanged)

Every input, derived state, control action, and correction is an event; state is only
ever a fold/projection of the log. Resume, handoff, and rollback are the same
fold-the-log mechanism. P0 mandates **event sourcing, not Kafka**.

## D9 — the log is swappable; Postgres first; Kafka scrapped (DECIDED 2026-07-29)

- **LogStore trait:** append (idempotent, content-hash event_id), keyed ordering,
  replay-from-offset, compaction-or-equivalent, watermark ("N behind head" — the free
  staleness metric).
- **Postgres backend first** (message-db pattern): zero new deployables on bc-prod;
  projections are tables. **Redis (Streams)** is a candidate second backend.
  **Kafka-API engines** (Redpanda/AutoMQ) are a documented escape hatch only —
  Terra: "we probably will never need Kafka for this."
- **Guard against the shim calcifying:** a conformance suite runs against ≥2
  implementations (in-memory + Postgres) from day one; no backend quirks may leak
  into folds.
- Rejected for the substrate: WarpStream (can't self-host), NATS/Pulsar (no
  compaction semantics), Temporal/Restate/DBOS (private truncated logs betray P0 —
  pattern donors only). Evidence:
  `../../research/software-garden-2026-07/notes/research-round2-substrate-ops-weight.md`.

## Envelope schema (v1, GP1.3)

- `schema_version` from day one (the garden's no-version regret, topic 12 §8).
- **Reserved day-one fields** so identity never needs a migration (decided direction,
  DC-10): `producer_principal{id,kind,runtime}` (OTel GenAI attribute names),
  `originating_human` with reserved `mac`, reserved `signature{key_id,scheme,sig}`;
  rule: signatures sign the canonical event_id.
- **Not adopted:** Buzz-style keypair-per-agent / signature-per-event — audit theater
  in a single-cluster trusted-runtime deployment; impersonation control belongs at
  the transport/broker layer (per-agent principals + ACLs). If anything is signed
  early, it's decisions (ApprovalGranted/PolicyDecision).

## Runtime (D10, DECIDED 2026-07-29)

Agents run on **Claude Code / Claude Agent SDK**, single-user, on Claude subscription
plans. SDK provides sessions/resume, per-run `max_turns`/`max_budget_usd`, and
PreToolUse hook enforcement. Rails: loop detection (repeated tool-call fingerprint,
ABAB), session caps (~50 turns/10 min/500K tokens), velocity alarms from metering
events; graceful cutoff = wrap-up note at ~90%, halt+persist+escalate at 100%.

**Open:** garden service language — TypeScript vs Python ([[07-open-questions]]).

Provenance: `../../research/decisions.md` P0/D9/D10; work:
`../../research/R1-WORK-BREAKDOWN.md` GP1/GP2.
