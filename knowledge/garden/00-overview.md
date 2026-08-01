---
doc_type: knowledge
topic: Software Garden — what we're building, for whom, constraints (R1 overview)
tags: [garden, r1, release, use-case, constraints, overview]
created: 2026-08-01
updated: 2026-08-01
status: current — R1 in flight, ships 2026-08-16; nothing built yet as of 2026-08-01 (design + work breakdown complete)
summary: The R1 use case, goal, and constraint set. Start here. Decided items cite decision ids; see 06-decisions-index.md for the full list.
---

# Software Garden — R1 overview

## What we're building (DECIDED: R1 2026-07-29, D13/D14 2026-08-01)

The Software Garden runs **4-5 apps on bc-prod mostly on autopilot**: agents own the
full inner loop — development, research, QA, optimization, user testing — inside a
release-engineering safety structure (ephemeral envs → preview → canary → prod, with
prod promotion human-gated). A **junior engineer** supervises: design reviews, feature
demos, prod promotions, incident response, and optional pair-programming drop-ins.
**Ships 2026-08-16** as a single-user pilot.

"Usable" means: the junior can see per-app status from live-derived cards, promote to
prod from evidence, resume/kill agents, declare incidents, and judge whether agent
spend/behavior is normal — without senior attention on the happy path.

## What Greenwood is now (DECIDED: D11)

Greenwood — the full Kafka event-sourced agent bus (see `../ARCHITECTURE.md`) — is an
**idea source, not the deliverable**. R1 ships its ideas pragmatically: event sourcing
(P0), attestations + evidence ledger (D12), per-message policy, feedback learning.
The full spec stays as reference for when real problems demand it.

## Constraints

1. **Date trades against scope, never against trust** — Aug 16 is self-imposed;
   scope demotes explicitly at pre-agreed cut lines (mid-point check 2026-08-06).
   Week-one junior trust is the unrecoverable resource (≥90% patrol precision,
   approval load = promotions/exceptions only).
2. **Ops weight ≤ one notch above kploy** — no platform team. Hence Postgres event
   log, zero new deployables (D9).
3. **Compose existing surfaces** — Slack/pin/bithub/kploy; no new UI apps. Agents
   mutate apps only via the kploy/GitOps paved road.
4. **Pilot harness = Claude Code / Agent SDK**, single-user, Claude subscription
   plans (D10).
5. **Safe defaults on, everywhere** — the observed anti-patterns in
   `../../research/topics/12-garden-hygiene-antipatterns.md` are a standing checklist.
6. **Human-only approvers** — no automated identity ever holds the checker role.
7. **Linear untouched** until Terra explicitly authorizes;
   `../../research/R1-WORK-BREAKDOWN.md` is the ticket source.

## Competitive frame (verified 2026-07-29)

No direct competitor does safe agent *operations* for non-expert operators: Buzz
(Block) is a collaboration substrate with no ops story; Kasava does PRDs; Efecto is a
zero-governance design canvas. Governance-by-default is the uncontested wedge.

Related: [[01-autonomy-model]], [[02-trust-attestations]], [[06-decisions-index]],
[[07-open-questions]].
