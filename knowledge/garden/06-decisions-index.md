---
doc_type: knowledge
topic: Decisions index — P0, R1, D1–D14 one-liners with status
tags: [decisions, adr, index, provenance]
created: 2026-08-01
updated: 2026-08-01
status: current — index only; full rationale lives in research/decisions.md (append-at-top log)
summary: Every decision, one line, with date and status. If a claim elsewhere conflicts with this index, this index wins; if this index conflicts with research/decisions.md, that file wins.
---

# Decisions index

Authority chain: `../../research/decisions.md` (full ADR log) > this index > any
other page. Statuses: **ACTIVE** (governs R1), **REFERENCE** (Greenwood-spec tier,
per D11), **SUPERSEDED**.

| Id | Decision (one line) | Date | Status |
|---|---|---|---|
| P0 | Event sourcing end-to-end; state is only ever a fold of the log | 2026-06-30 | ACTIVE |
| R1 | First usable garden ships 2026-08-16; junior operator, 4-5 apps | 2026-07-29 | ACTIVE |
| D1 | Hybrid claims: agent proposes structure+provenance, bus establishes trust | 2026-06-30 | REFERENCE (idea lives in D12) |
| D2 | Handoff via graduated synthesis → claims → evidence, lazy expansion | 2026-06-30 | REFERENCE (survives as spawn-brief pattern) |
| D3 | Async confirmation-as-event + boundary gate | 2026-06-30 | REFERENCE |
| D4 | Hermetic replay evals (recorded tool results as fixtures) | 2026-06-30 | REFERENCE (idea lives in GP8 eval-builder) |
| D5 | Graft harness-adapter protocol | 2026-06-30 | REFERENCE (deferred; single-harness pilot per D10) |
| D6 | Backing store deliberation (Kafka-era) | 2026-06-30 | SUPERSEDED by D9 |
| D7 | Bus-side decomposition; mechanical provenance; classifier triage | 2026-07-02 | REFERENCE (mechanical-provenance idea lives in D12's evidence ledger) |
| D8 | Messaging is the only inter-agent primitive; governance per-message | 2026-07-02 | ACTIVE (v1 body: message events + static policy + brief screen) |
| D9 | Swappable LogStore; Postgres first; Kafka scrapped for now | 2026-07-29 | ACTIVE |
| D10 | Pilot on Claude Code/Agent SDK, single-user, subscription plans | 2026-07-29 | ACTIVE |
| D11 | Greenwood is an idea source, not the deliverable; per-message governance, verification, and learning ship in R1 pragmatically | 2026-07-29 | ACTIVE |
| D12 | Attestations v1: atomic claims validated against a runner-captured evidence ledger; no-evidence-ref ⇒ UNSUPPORTED | 2026-07-29 | ACTIVE |
| D13 | Mostly autopilot: autonomy by environment (rings), prod promotion human-gated, app modes standardize/build/run | 2026-08-01 | ACTIVE |
| D14 | Autonomous inner loop (dev/research/QA/optimization/user-testing); humans at meaning boundaries (design review, demos, client surfaces, pair drop-in) | 2026-08-01 | ACTIVE |

Also binding: **approval semantics rulings** (2026-07-29, evidence-backed, in
`r1-spec-inputs.md` §F): human-only approvers; maker-checker aggregate; tier law =
min(agent ceiling, human ceiling); TTL'd parameter-bound approvals.

Pending Terra: DC-2 (Cedar formally), DC-5/DC-9 default numbers as versioned policy,
and everything in [[07-open-questions]].
