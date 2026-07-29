---
doc_type: topic
topic: Human feedback capture & propagation — learning over time as eventual consistency
project: Greenwood
tags: [feedback, learning, evals, eventual-consistency, r1, p9, close-out-notes]
sources: [Terra 2026-07-29, WORK-BREAKDOWN P9, software-garden-2026-07 research sweep]
created: 2026-07-29
updated: 2026-07-29
status: active — design thinking from Terra's 2026-07-29 framing; R1 capture proposal pending Terra
summary: Terra's framing - the goal is not immediate perfection but learning and improving, i.e. eventual consistency. Design consequence - R1 must ship feedback CAPTURE (cheap, greenfield-critical, unretrofittable); the learning machinery (P9/M4) folds it later. Every human action in the system is dual-purpose - an operational act AND a training label.
---

# Human feedback: capture & propagation as eventual consistency

**Terra's framing (2026-07-29, verbatim intent):** how do we capture and propagate
feedback so the system learns and improves over time? The goal isn't immediate
perfection — it's learning and improving: *eventual consistency*.

## 1. Why this framing fits Greenwood exactly

Greenwood is already an eventually-consistent system: state is a fold of the log, and
projections converge as events propagate. Terra's framing makes feedback a first-class
citizen of that same model rather than a bolted-on "ML feedback loop":

- **Feedback is just more events** (P0). A correction, a denial, a thumbs-down, an
  edited plan — each is an event with provenance (`derived_from` → the thing being
  judged). This is what T9.1 already specifies.
- **Learning is re-folding with better parameters.** Policies, eval suites, docs, and
  classifiers are projections over the feedback stream. They converge toward correct;
  they are never synchronously perfect. Conflicting feedback resolves the way
  conflicting events always do: later, with full history, by fold rules — not by
  blocking the operator in the moment.
- **Convergence, not perfection, is the acceptance criterion.** The system is allowed
  to be wrong today if today's wrongness reliably becomes tomorrow's eval case,
  policy adjustment, or doc fix. What is NOT allowed is wrongness that leaves no
  trace — that's divergence.

## 2. The one design principle: every human action is dual-purpose

Don't build a feedback feature. Make every operational human touchpoint emit its
label as a side effect:

| Human action (operational) | Feedback signal (label) | Propagates to |
|---|---|---|
| Approve at a gate | positive label on the proposal shape | policy register (autonomy promotion evidence) |
| **Deny at a gate + required reason code** | the richest cheap label: a bad proposal, with why | eval case (T9.2), triage classifier (T5.5), policy tightening |
| **Edit-before-approve** (junior modifies the proposal, then approves) | a paired (bad, good) example — the strongest supervision available | eval + harness refinement (T9.4) |
| Break-glass invocation | gate miscalibration signal | tier-matrix review (frequency is a gate-quality metric — round-2 break-glass work) |
| Patrol finding dismissed / accepted / snoozed | per-check precision label | per-check graduation & retirement (patrol charter) |
| Thumbs on a navigator answer + correction | KB defect with locus | KB fix + eval case |
| Close-out note on task completion | outcome ground truth + rationale | KB (teach-don't-deskill channel), failure clustering (T9.6) |
| Implicit: test failure, rollback, task re-opened, `TrustTransition{REJECTED}` | free negative labels, no human effort | eval builder (T9.1 already wires these) |

Two rules make the table work:
1. **Harvest natural actions; never ask for ratings.** Feedback fatigue is the same
   disease as approval fatigue. The junior should almost never do something *extra*
   to give feedback — the exceptions (reason-on-deny, one-line close-out) are
   workflow-enforced at moments where the context is already loaded (topic 12 §3:
   push-based protocols that rely on remembering go unused).
2. **Reasons are enums-plus-freetext, not essays.** A deny without a reason code is
   half a label; a mandatory essay kills compliance. Reason codes also fix the label
   noise the corrected mis-calibration finding predicts (arXiv 2602.00496: novices
   over-rely AND over-avoid — raw approve/deny rates are therefore noisy; the reason
   code disambiguates "denied because wrong" from "denied because scared").

## 3. Capture now, learn later (the R1 / M4 split)

The same logic as the envelope identity reservations (r1-spec-inputs §G): **you
cannot retroactively capture signals you didn't record, but you can always add
learning machinery later.** So:

- **R1 ships CAPTURE** (cheap, mostly schema + two workflow rules):
  - `Feedback` event schema (T9.1's shape) + `derived_from` provenance — even though
    nothing consumes it yet.
  - Reason-required-on-deny (enum + optional freetext) at every T3/T4 gate.
  - Edit-before-approve recorded as proposal-diff on the approval event.
  - One-line close-out note enforced at task completion.
  - Patrol finding lifecycle states (dismissed/accepted/snoozed) as events.
  - Break-glass invocations already land as events (round-2 design).
- **M4 ships LEARNING** (P9 as designed — eval-builder, hermetic replay, harness
  versioning, regression suite, failure clustering; T5.5 classifier retraining).
  Nothing in R1 blocks on it; everything in R1 feeds it.

## 4. Propagation surfaces (each one a projection)

1. **Evals** — feedback → frozen-context EvalCase → regression suite (P9). The
   flagship path; already designed.
2. **Policy register** — approve/deny/break-glass history per action-type is the
   evidence for autonomy promotion/demotion (the ITIL standard-change mechanism from
   round-1 research: autonomy is *earned per action-type on evidence, granted by
   humans*). Eventual consistency in governance: tiers converge to the right
   strictness.
3. **Knowledge base** — close-out notes and answer-corrections append to the KB with
   provenance; staleness machinery (two-date frontmatter) governs their decay. The
   KB converges toward true the same way projections do.
4. **Triage classifier** — `TrustTransition` + deny-reasons are labeled training data
   (T5.5's pipeline); new versions gate through Coldframe before promotion — learning
   itself passes an approval gate, which closes the loop honestly.

## 5. Failure modes to design against

- **Goodhart:** optimizing thumbs/approval-rate instead of outcomes. Counter: implicit
  outcome signals (tests, rollbacks, re-opens) outweigh explicit sentiment in eval
  selection; Coldframe gates promotion on replayed outcomes, not satisfaction.
- **De-skilling via passive acceptance:** an approval with no engagement is a weak
  label AND a skill-formation failure. Counter: evidence-rich cards make engagement
  the path of least resistance; edit-before-approve is celebrated in the UX, not
  discouraged.
- **Label noise from bidirectional mis-calibration** (corrected finding): reason codes
  + treating junior labels as *weak* supervision to be confirmed by outcomes, not
  gospel.
- **Feedback that mutates nothing visible:** if the junior's denials never visibly
  change behavior, they stop giving them. Counter: a "your feedback did this" surface
  (even a monthly digest: N denials → M policy changes → K eval cases) — the
  propagation must be legible to its source.

## 6. Open questions (for Terra / future passes)
- Reason-code taxonomy for denials (start tiny: wrong-target, too-risky, bad-timing,
  insufficient-evidence, other+text?).
- Does edit-before-approve capture belong in the approval aggregate (event payload) or
  as a linked `Feedback` event? (Leaning: payload — it's part of the approval's
  meaning.)
- Weighting junior labels vs outcome labels in eval selection (M4 question, record now).
- Whether the "feedback did this" digest is R1 or M4 (leaning M4 — but the events that
  make it possible are R1 capture).
