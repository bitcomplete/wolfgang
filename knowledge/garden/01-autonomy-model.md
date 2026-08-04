---
doc_type: knowledge
topic: Autonomy model — environment=risk, rings, app modes, human engagement surfaces
tags: [autonomy, autopilot, rings, canary, promotion, app-modes, human-gates, d13, d14]
created: 2026-08-01
updated: 2026-08-01
status: current — DECIDED (D13 + D14, 2026-08-01); implementation is R1 work (GP10/GP11)
summary: Autonomy is granted by environment, not per-action approval. Below-prod rings are fully autonomous behind automated gates; prod promotion is human-gated. Apps carry modes standardize|build|run. Humans engage at meaning boundaries - design review, demos, client surfaces, incidents, pair drop-in.
---

# Autonomy model (DECIDED: D13, D14 — 2026-08-01)

## The principle

**Autonomy by environment, not per-action approval.** Safety is structural — release
engineering, not instructions or approval queues: ephemeral dev environments, preview
environments, canary releases, ringed (risk-gated) deployment, continuous deployment,
environment promotion. Machines own *mechanical* boundaries; humans own *meaning*
boundaries.

## Rings (promotion path)

`ephemeral → preview → canary → prod`

- **Below prod: fully autonomous.** Blast radius contained by construction. Every
  ring transition passes **automated** gates: tests green, attestations CONFIRMED
  (see [[02-trust-attestations]]), canary metrics, conformance, automated
  user-testing results from preview.
- **Prod promotion: green-by-construction, system-level human gate (D17,
  2026-08-03).** Release engineering owns promotion: the pipeline promotes when
  mechanically green (release attestation COMPLETE per D16, independent review
  verdicts per D15, canary armed). The junior handles **the system, never the
  details** — a one-tap system-level card; exceptions and non-green releases route to
  them; a promotion card requiring diff-reading is a design bug. Canary +
  auto-rollback manufactures reversibility on the prod side.
- **Standing irreversibles stay human-gated regardless of ring:** data mutation,
  secrets, tier-4 (two-person) actions.
- Every agent **assumes and depends on** the app having ephemeral dev environments —
  a hard per-app dependency (built on kploy `preview:` machinery).
- Ops actions in run mode (proposed reading, NOT yet confirmed by Terra): reversible
  with registered rollback = autonomous + logged; irreversible = human-gated.

## The fully autonomous inner loop (D14)

Development, research, QA, optimization, and user testing run without human
involvement, inside the rings above.

## Human engagement surfaces (D14)

| Surface | Purpose |
|---|---|
| **System design review** (weekly, from auto-generated design deltas) | Reduce cognitive debt — keep the humans' mental model current; comprehension debt is the #1 agent-era debt |
| **Feature demos** (auto-generated in the feature's preview env) | Outcome verification at the human level; accept/redirect feeds the learning loop |
| **Client meetings / materials** | Human-fronted, garden-supplied (release notes derive from the event log) |
| **Prod promotion** | The operational human gate |
| **Incident response** | Incident mode / break-glass — see [[02-trust-attestations]] |
| **Pair-programming drop-in** | For tricky/important features: join any brief as an interactive Claude Code session with garden context; hand back to autonomous mode; entry/exit are events |

## App lifecycle modes (D13)

`standardize | build | run` — an event-sourced registry field per app.

- **Run mode requires conformance** to the App Operations Contract (kploy-only
  deploys, bc-prod.yaml, sealed secrets, health endpoints, card+runbook, ephemeral +
  preview env support, canary-capable config) — patrol-verified, mechanically
  enforced.
- **Standardize mode** is agent-assisted onboarding: gap report → standardization
  briefs → re-check → promote to run mode. It is the permanent funnel for every
  future app.
- R1 target: ≥2 apps in run mode by 2026-08-16; the rest ship in standardize mode
  honestly.

## Why this is evidence-sound

The 2025-26 practitioner findings (only ~8% run agents autonomously in prod; co-pilot
preference) warn against *unstructured* prod autonomy. Here autonomy lives below prod
behind automated gates, and humans hold exactly the gate the evidence demands —
irreversible world impact. Enforcement is structural (environments, credentials,
pipelines), never instructional — the Replit-incident lesson.

Provenance: `../../research/decisions.md` D13/D14; work:
`../../research/R1-WORK-BREAKDOWN.md` GP10/GP11.
