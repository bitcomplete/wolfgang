---
doc_type: knowledge
topic: Trust — attestations, evidence ledger, hallucination catching, approvals, break-glass
tags: [attestations, hallucinations, verification, evidence-ledger, approvals, maker-checker, break-glass, d12]
created: 2026-08-01
updated: 2026-08-01
status: current — DECIDED (D12 2026-07-29 + approval semantics rulings); implementation is R1 work (GP3/GP5)
summary: Agents attest results as atomic claims; validation runs per-attestation against a runner-captured evidence ledger agents cannot author. No evidence ref = UNSUPPORTED automatically. Human-only approvers; maker-checker aggregate; incident-scoped break-glass.
---

# Trust: attestations & verification (DECIDED: D12 — 2026-07-29)

## The hallucination problem and its answer

Concern: agents hallucinate results ("tests pass", "file deployed"). Answer — a
**thin claims substrate**:

1. **Attest, don't extract.** Every RESULT must carry atomic attestations
   `{id, statement, kind: test-passed|file-changed|deploy-done|probe-observed|inference,
   evidence_refs[]}` as enforced structured output. Zero attestations → result
   rejected mechanically. (We control the report-back schema, so the full spec's
   free-text decomposer is unnecessary.)
2. **The evidence ledger is runner-captured, never agent-authored.** Hooks record
   tool results, exit codes, diffs, probe outputs as events. Agents cannot forge the
   record they're checked against (mechanical provenance — the D7 insight).
3. **Validation per-attestation, cheapest first:** deterministic checks by kind
   (ref dereferences, exit code matches, diff exists) are hallucination-*proof*;
   model entailment (fresh context) for inference-kind claims only. Automatic rule:
   **no dereferenceable evidence ref ⇒ UNSUPPORTED, zero model calls** — catches
   phantom test runs, fabricated files, invented deploys.
4. **Verdicts govern composition:** spawn briefs compose from CONFIRMED attestations
   only by default; UNSUPPORTED results cannot be approved without a logged,
   reason-coded override. Ring gates require CONFIRMED attestations
   ([[01-autonomy-model]]).
5. **Verifier fallibility → the learning loop, not more verifiers:** sampled audits
   of CONFIRMED verdicts become eval cases ([[04-feedback-learning]]).

## Per-message policy (D8's idea, v1 body)

Every inter-agent exchange (SPAWN_BRIEF / RESULT / STATUS) is a Message event on the
log — the only content path between agents. Static declarative rules rule
deliver/screen; rulings are logged, versioned-as-events, replay-deterministic. Spawn
briefs get a cheap-model screen against cited sources.

## Approvals (rulings 2026-07-29, evidence-backed)

- **Only humans approve** — no automated identity holds the checker role (industry
  position: Entra PIM bars service principals; GitHub reviewers are humans/teams).
  `checker != maker`; at tier-4, `checker != originating_principal`.
- **Maker-checker aggregate** (fintech shape): parameter-bound, TTL'd,
  edit-invalidates-approval; approval is an event the decider requires ("gate in the
  fold" — externally grounded by Faramesh/Zylos).
- **Tier law:** effective authority = min(agent ceiling, originating-human ceiling);
  no delegated action exceeds what its initiating human could authorize.
- Evidence-rich cards always — what/why/evidence/rollback, never bare "approve?".

## Incident mode / break-glass (proposal accepted into R1 scope, DC-11)

Declaring an incident (`IncidentDeclared` event — one command, never silent) pages
Terra <60s but the junior never waits; a **pre-vetted emergency set** runs at
reversible-tier for that app (kploy rollback-to-last-good, restart/scale, kill
switches, freeze) — never data mutation/secrets. 60-min TTL or incident close;
blameless review within 72h; frequent use ⇒ recalibrate gates, never reprimand.
Kill switches (pause-agents, freeze-deploys) are the emergency floor.

Provenance: `../../research/decisions.md` D12;
`../../research/software-garden-2026-07/r1-spec-inputs.md` §F/H;
work: `../../research/R1-WORK-BREAKDOWN.md` GP3/GP4/GP5.
