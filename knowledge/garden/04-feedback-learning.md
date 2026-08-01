---
doc_type: knowledge
topic: Feedback & learning — capture everywhere, propagate as eventual consistency
tags: [feedback, learning, evals, regression, eventual-consistency, close-out, reason-codes]
created: 2026-08-01
updated: 2026-08-01
status: current — DECIDED (D11.3 2026-07-29 - learning ships in R1, capture alone insufficient); implementation is R1 work (GP8)
summary: Every human action is dual-purpose - operational act AND training label. R1 ships capture (reason-on-deny, edit-diffs, close-outs, demo accept/redirect) AND a working loop - feedback freezes into replayable eval cases; the regression set blocks harness changes; propagation is legible.
---

# Feedback & learning (DECIDED: D11.3 — "capture only isn't learning")

## The frame (Terra, 2026-07-29)

The goal isn't immediate perfection — it's learning and improving: **eventual
consistency**. The system may be wrong today if today's wrongness reliably becomes
tomorrow's eval case, policy change, or doc fix. Wrongness that leaves no trace is
the only forbidden failure.

## Principle: every human action is dual-purpose

Operational act AND training label — harvest natural actions, never ask for ratings:

| Action | Label | Propagates to |
|---|---|---|
| Deny + required reason code | richest cheap label | eval case, policy tightening |
| **Edit-before-approve** (diff recorded) | paired bad/good example — strongest supervision | eval + harness refinement |
| Demo accept/redirect (D14) | outcome label | new brief with provenance / eval |
| Break-glass invocation | gate miscalibration signal | tier recalibration |
| Patrol fixed/not-useful tap | per-check precision label | check graduation/demotion |
| Close-out note (workflow-enforced, one line) | outcome ground truth | KB, failure clustering |
| Implicit: test fail, rollback, UNSUPPORTED verdict, re-open | free negative labels | eval builder |

Reason codes are enums+optional-text (v1 proposal: wrong-target / too-risky /
bad-timing / insufficient-evidence / other) — they also disambiguate the verified
finding that juniors mis-calibrate in *both* directions ("denied because wrong" vs
"denied because scared").

## The loop that actually turns in R1 (GP8)

1. **Capture** — the table above, as events with `derived_from` provenance.
2. **Eval-case builder** — a denial/correction/UNSUPPORTED verdict freezes the brief +
   context refs into a replayable case (SDK session replay); must reproduce the
   failure.
3. **Regression set** — accumulated cases run *before* any harness/prompt/policy
   change; red blocks the change (human-overridable, logged). A fixing tweak flips
   red→green.
4. **Legible propagation** — weekly "your feedback did this" digest (N denials → M
   changes → K eval cases). If feedback visibly changes nothing, humans stop giving
   it.
5. **Policy-evidence register** — approve/deny/break-glass history per action-type is
   the evidence for tier promotion/demotion proposals (ITIL standard-change shape);
   recalibration is Terra-approved, event-logged.

## Guardrails on the loop

- **Goodhart:** outcome signals (tests, rollbacks) outweigh sentiment in eval
  selection; promotions gate on replayed outcomes.
- **De-skilling:** design review + demos + close-outs make the system teach
  ([[01-autonomy-model]] human surfaces); junior labels are weak supervision to be
  confirmed by outcomes, not gospel.
- The verifier itself is inside the loop: sampled audits of CONFIRMED verdicts become
  eval cases ([[02-trust-attestations]] §5).

Provenance: `../../research/topics/13-human-feedback-eventual-consistency.md`;
work: `../../research/R1-WORK-BREAKDOWN.md` GP8.
