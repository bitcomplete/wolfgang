---
doc_type: knowledge
topic: Open questions — what is NOT decided (never answer these as settled)
tags: [open-questions, pending, terra]
created: 2026-08-01
updated: 2026-08-01
status: current — check research/decisions.md for rulings newer than `updated` above
summary: The live open queue. A navigator answer touching any of these must say OPEN and cite this page, not improvise a ruling.
---

# Open questions (as of 2026-08-01)

## Blocking ticketing / build start
1. **Garden service language: TypeScript vs Python** (D10 re-opened this — the agent
   runtime is the SDK, so the earlier Rust lean no longer follows). Needs Terra.
2. **Name the 4-5 apps** — and which already have kploy `preview:` support (sizes
   both the onboarding work and the run-mode-by-Aug-16 subset). Needs Terra.

## Needs Terra confirmation (proposals exist)
3. **Run-mode ops actions under D13:** proposed reading — reversible with registered
   rollback = autonomous+logged; irreversible = human-gated. Confirm/correct.
4. **Design-review "significant change" triggers** (GP11.1 proposes: new dependency,
   schema/API change, cross-app pattern, security surface) and the weekly cadence.
5. **Deny reason-code taxonomy v1** (proposed: wrong-target / too-risky / bad-timing /
   insufficient-evidence / other+text).
6. **Per-app emergency action sets** for incident mode (contents of the pre-vetted
   T2 set per app).
7. **Approval TTL defaults** as versioned policy (proposed: T3 4h pending/15min
   validity; T4 24h/60min; credentials ≤8h) — granted-but-unexecuted validity has no
   shipped precedent anywhere; these are design-call numbers.
8. **Cedar as the policy engine** (DC-2) — recommended, not formally ruled.

## Standing process guardrails (not questions, never violate)
- **Linear untouched** until Terra explicitly authorizes; ticket source is
  `../../research/R1-WORK-BREAKDOWN.md`.
- Scope demotes only at pre-agreed cut lines (mid-point check **2026-08-06**), never
  silently.

## Proposed R1 schema tweaks from the 2026-08-03 ideation (topic 14, pending Terra)
- UNEXPECTED_FINDING / replan message kind; attestation refs may reference other
  attestations (compositional drill-down); golden-signal names in rollback criteria;
  user-journey registry in the App Operations Contract (R1-if-cheap).

## Watch items
- Buzz's `buzz-workflow` approval-gate roadmap (overlap grows if it ships ops
  workflows).
- R2 candidates parked in the reference bank: claims decomposition at scale, triage
  classifier, rewind, multi-harness grafts, multi-user/RBAC, semantic doc-drift +
  cost-anomaly patrol checks, automated tier promotion.
- R2 candidates from the 2026-08-03 ideation (`research/topics/14-*.md`): swarm-state
  statistical baselines (Markov/HMM framing — attestation verification is the
  observation-correction step), dependency-aware segment-scoped prod rollout,
  standing synthetic-monitoring agents, automated re-planning loops. Research-tier:
  categorical verification; genealogy governance on cyclic coordination graphs.
