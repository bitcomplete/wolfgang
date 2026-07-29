---
doc_type: work-breakdown
topic: R1 — software garden v1 (2026-08-16), the ACTIVE ticket source
project: software-garden
tags: [work-breakdown, r1, release, tickets, garden]
sources: [decisions R1/D9/D10/D11, r1-spec-inputs DC-1..13, software-garden-2026-07]
created: 2026-07-29
updated: 2026-07-29
status: active — supersedes WORK-BREAKDOWN.md as the ticket source for R1 (that file is now Greenwood reference architecture per D11); do NOT create in Linear until Terra authorizes
summary: The Aug-16 garden v1 decomposed into buildable units. Frame - junior engineer + agents operate 4-5 apps on bc-prod, single-user pilot on Claude Code/Agent SDK (D10), Postgres event log behind a swappable trait (D9), Greenwood ideas delivered pragmatically (D11) - gates, per-message policy, evidence-checked verification, and a working feedback-learning loop. Evidence citations point into research/software-garden-2026-07/r1-spec-inputs.md (DC-1..13) and the round-2 notes.
---

# R1 Work Breakdown — Software Garden v1 (ship 2026-08-16)

**Goal (decisions.md R1):** the junior can supervise agent-assisted operations of the
4-5 apps — see status, approve/deny risky actions from evidence, resume/kill agents,
declare incidents, stay within cost visibility — without senior attention on the
happy path. Single-user pilot (D10). Every safety default on (topic 12 §4).

**Design stance (D11):** Greenwood ideas, pragmatic bodies. Event sourcing stays the
spine (P0) on a swappable log (D9). No claims substrate, no Kafka, no multi-harness,
no new UI applications — compose Slack/pin/bithub/kploy.

**Milestone gates:**
- **R1a — Rails floor (Aug 8):** gates + rails + incident mode live on one app.
- **R1b — Garden loop (Aug 13):** all apps onboarded; per-message policy, verifier
  report-back, patrol shadow→live, feedback loop closing.
- **R1c — Pilot-ready (Aug 16):** game-day drill passed with the junior; runbooks
  verified-by-execution; week-one protections armed.
- Mid-point scope check **Aug 6**: anything at risk demotes per the pre-agreed cut
  lines (marked ✂ below), never by silent slippage.

---

## GP1 — Event log & folds (Postgres behind LogStore trait) · R1a
Refs: D9, P0, substrate note. Language per D10's open question (Terra: TS vs Python).

- **G1.1 — LogStore trait + in-memory impl + conformance suite.** Why: D9's guard —
  no backend quirks in folds. Scope: append (idempotent, content-hash event_id),
  keyed ordering, replay-from-offset, compaction-equivalent, watermark ("N behind
  head" — the free staleness metric, C.1). Acceptance: conformance suite green on
  in-memory impl; fold determinism test (same events → same state, twice).
- **G1.2 — Postgres backend (message-db pattern).** Acceptance: conformance suite
  green; runs on bc-prod Postgres with zero new deployables.
- **G1.3 — Event envelope v1 (Protobuf or typed JSON).** Scope: schema_version from
  day one (topic 12 §8); reserved fields per DC-10 — `producer_principal{id,kind,
  runtime}` (OTel GenAI names, DC-4), `originating_human` (+reserved mac), reserved
  `signature`; rule recorded: signatures sign the canonical event_id. Acceptance:
  round-trip + schema-evolution test.
- **G1.4 — Projections: session state, per-app status, metering.** Why: cards, cost
  tiles, and resume all fold from here. Acceptance: projections rebuild from replay;
  watermark surfaces in every rendered view (staleness honesty, C.1/C.2).

## GP2 — Agent runtime on Claude Agent SDK · R1a
Refs: D10, B.1, cost note.

- **G2.1 — Runner: SDK session ↔ event log.** Scope: every run emits
  session/turn/tool events; SDK session id ↔ lineage root; single-user auth.
  Acceptance: a trivial task's full trace folds from the log.
- **G2.2 — Rails config:** max_turns, `max_budget_usd` per task where metered, wall-
  clock cap, loop detection (repeated tool-call fingerprint 2-3× → halt; ABAB check)
  (B.5). Acceptance: seeded runaway halts with "stopped at limit + status note" (B.7).
- **G2.3 — Resume/kill as operator actions.** Scope: SDK session resume wired to a
  button/command; kill emits terminal event; both T2-tier. Acceptance: kill -9 the
  runner mid-task → resume completes it (topic 12 §6 acceptance).
- **G2.4 — Velocity/anomaly tile.** Scope: turns+tokens/hour from metering projection;
  "is this normal?" vs baseline (B.11 anchors). ✂ demotes to daily digest number.

## GP3 — Gates, approvals & incident mode · R1a
Refs: DC-1/2/3/8/9/11, F, H, A.4.

- **G3.1 — Tier policy engine (Cedar) at the tool boundary.** Scope: T1-T4 matrix as
  Cedar policies, default-deny, params in context; tier law = min(agent ceiling,
  originating-human max) (F.2). Acceptance: policy decisions logged as events with
  policy_version; replay-deterministic (F.4).
- **G3.2 — PreToolUse hook enforcement.** Why: bypass-proof today on the SDK (A.4);
  defense-in-depth under G3.1. Acceptance: veto holds under bypassPermissions.
- **G3.3 — Approval aggregate (maker-checker).** Scope: fintech state machine (F/G2);
  ApprovalGranted only from a human; checker≠maker; edit-invalidates-approval;
  parameter-bound, TTL'd (DC-9 defaults as versioned policy). Acceptance: expired/
  edited/self approvals all rejected by the fold.
- **G3.4 — Slack approval inbox with evidence cards.** Scope: what/why/evidence/
  rollback per card (never bare "approve?"); approve/deny(+reason code)/edit actions.
  Acceptance: card→decision→event round-trip < 1 min of junior effort.
- **G3.5 — Incident mode / break-glass (DC-11, H).** Scope: IncidentDeclared event;
  pages Terra <60s, junior never waits; pre-vetted emergency set runs at T2 for that
  app (kploy rollback-to-last-good, restart/scale, kill switches, freeze); 60-min TTL
  or incident close; blameless review artifact. Acceptance: drill passes the H
  metrics (first action <5 min, 100% reviewed).
- **G3.6 — Kill switches:** pause-all-agents, freeze-deploys (per-app + global).
  Acceptance: instant, reversible, event-logged.

## GP4 — Per-message governance v1 (the D8 idea, pragmatic) · R1b
Refs: D11.1, D8 (reference), F.4.

- **G4.1 — Message boundary events.** Scope: every inter-agent exchange (SPAWN_BRIEF,
  RESULT, STATUS) is a Message event on the log — the only way content crosses
  agents, even inside one SDK process. Acceptance: no agent-to-agent content path
  bypasses the log (code-review + test).
- **G4.2 — Static policy on messages (no ML).** Scope: declarative rules by
  type/sender/recipient → deliver/screen; rulings logged as PolicyDecision events;
  policy config versioned as events. Defaults: STATUS delivers; SPAWN_BRIEF screens.
  Acceptance: deterministic, replayable rulings.
- **G4.3 — Spawn-brief screen (cheap model).** Scope: one screening call checks a
  brief's assertions against its cited sources before delivery; block+feedback on
  failure; sampled audit of delivered messages (measured false-green rate).
  ✂ demotes to log-only (screen advisory, not blocking).

## GP5 — Attestations & verification v1 (the D1/D7 ideas, pragmatic; D12) · R1b
Refs: D12, D11.2, arXiv note (grounded-verifier findings), planning report slice contract.

- **G5.1 — Slice contract as spawn schema.** Scope: Why/Scope/Acceptance-with-
  verification-command/Deps/Tier/Budget/Rollback — every task brief carries it.
  Acceptance: runner refuses a brief without falsifiable acceptance.
- **G5.2 — Attestation schema (attest, don't extract).** Scope: RESULT messages must
  carry atomic attestations {id, statement, kind: test-passed|file-changed|deploy-done|
  probe-observed|inference, evidence_refs[]} as enforced structured output; a result
  with zero attestations is rejected mechanically. Acceptance: schema round-trip;
  free-prose-only result bounces back to the agent.
- **G5.3 — Evidence ledger (mechanical provenance).** Scope: runner captures tool
  results, exit codes, diffs, probe outputs as events via hooks — agents cannot write
  the ledger they are checked against (D7's insight, v1 form). Acceptance: an agent
  claiming an un-run test has no matching ledger entry, provably.
- **G5.4 — Per-attestation verifier, cheapest-first.** Scope: deterministic checks by
  kind (ref dereferences, exit code matches, diff exists) — hallucination-proof; model
  entailment (fresh context) only for inference-kind claims — hallucination-resistant;
  automatic rule: **no dereferenceable evidence ref ⇒ UNSUPPORTED, zero model calls**;
  per-attestation verdict events (evidence-anchored only — ungrounded correction
  destabilizes, arXiv 2606.27409). Acceptance: planted phantom-test, fabricated-file,
  and false-inference attestations are each flagged before the junior sees the card.
- **G5.5 — Verdicts govern composition + approval.** Scope: result cards render
  per-attestation verdicts + evidence links; spawn briefs compose from CONFIRMED
  attestations only by default (unconfirmed require explicit, flagged inclusion);
  UNSUPPORTED results can't be approved without a logged, reason-coded override;
  sampled audit of CONFIRMED verdicts feeds GP8 (verifier misses become eval cases).

## GP6 — Operator surface (P11) · R1a→R1b
Refs: C, cognitive-debt + hci reports, topic 12 §1-3.

- **G6.1 — Per-app card ×4-5:** status/version/last-deploy derived live (derive-don't-
  quote), watermark-stamped, two-date frontmatter on prose; skip-level overview.
- **G6.2 — Runbook per app, dual-audience:** diagnostic blocks + copy-paste agent
  prompts; steps bind to live dashboard queries (C.5); `verified` means executed.
  Acceptance: junior + agent each complete one drill from the runbook alone.
- **G6.3 — Staleness-honest navigator answers:** age/provenance stamps, warn/refuse
  tiers (dbt warn_after/error_after shape, C.4); derive-don't-quote for anything
  recomputable. Acceptance: mahdi-style 7-week-stale answer is impossible by
  construction (stamp or refusal).
- **G6.4 — App Operations Contract + onboarding the 4-5 apps:** kploy-only deploys,
  bc-prod.yaml provisioning, sealed secrets, /healthz+/readyz+/metrics, card+runbook
  present. Acceptance: conformance check green per app (feeds GP7).
- **G6.5 — Close-out notes, workflow-enforced** (one line, at completion; feeds GP8).

## GP7 — Patrol v1 (DC-13 charter) · R1b
Refs: I, patrol note.

- **G7.1 — Five deterministic checks:** health probes (retry ×3, tri-state), deploy-
  vs-repo drift, git hygiene >7d, runbook contract (verified-TTL + live-links
  dereference), cert/backup. Every finding: evidence link + attached fix.
- **G7.2 — Lifecycle + digest:** fingerprint dedupe, auto-close-when-fixed, typed
  expiring exemptions via PR; one daily per-app digest (all-clear = one line);
  closed interrupt class only for cert<7d/backup-missing/healthz-down.
- **G7.3 — Shadow-first graduation:** shadow to Terra; graduate per (check,app) after
  7 clean days + seeded-fault firing test; ≥90% rolling effective precision via
  one-tap fixed/not-useful; 2 strikes in 14 days → auto-demote. Anything unready
  Aug 16 ships in shadow. ✂ any check may ship shadow-only.

## GP8 — Feedback → learning loop (P9-lite; D11.3) · R1b
Refs: topic 13, DC-12, P9 (reference).

- **G8.1 — Capture set:** Feedback events with derived_from; reason-code-on-deny
  (taxonomy v1: wrong-target / too-risky / bad-timing / insufficient-evidence /
  other+text); edit-before-approve diff on the approval event; close-out notes;
  patrol not-useful taps. Acceptance: every table-row action in topic 13 §2 emits
  its label with zero extra junior steps beyond reason-on-deny + close-out.
- **G8.2 — Eval-case builder (manual-trigger ok):** a denial/correction/UNSUPPORTED
  verdict freezes the brief + context refs into a replayable eval case (SDK session
  replay). Acceptance: a thumbs-down becomes a case that reproduces the failure.
- **G8.3 — Regression set + pre-change run:** harness/prompt/policy changes run the
  accumulated cases first; red blocks the change (human-overridable, logged).
  Acceptance: a fixing tweak flips a case red→green; the suite runs in CI.
- **G8.4 — "Your feedback did this" digest (weekly):** N denials → M policy/harness
  changes → K eval cases; legible propagation (topic 13 §5). ✂ demotes to monthly.
- **G8.5 — Policy-evidence register:** approve/deny/break-glass history per
  action-type — the evidence base for tier promotion/demotion proposals (ITIL
  standard-change mechanism). Recalibration is Terra-approved, event-logged.

## GP9 — Pilot readiness · R1c
- **G9.1 — Week-one protections:** patrol strikes armed, approval queue budget
  (handful/day — exceeding it is a design bug to fix, not a junior problem), high
  patrol thresholds, first-day path (one app, one runbook, one approval).
- **G9.2 — Game-day drill with the junior (before Aug 16):** incident declare →
  emergency action → auto-verify → review; plus one resume and one deny-with-reason.
  Acceptance: H's drill metrics + junior can narrate what happened from the cards.
- **G9.3 — R1 retro + R2 seed:** what demoted at Aug 6, break-glass/gate/patrol
  metrics baseline, feedback-loop counts; feeds the R2 scope decision.

---

## Deferred (reference bank — pull when a problem demands, per D11)
Full claims substrate & triage classifier (P4/P5), full per-message cascade & rewind
(P6), swarm/fan-out (P7), Kafka-API engines (D9 escape hatch), grafts/multi-harness
(P10), multi-user/RBAC, SPIFFE/per-agent keypairs, semantic doc-drift + cost-anomaly
patrol checks, automated tier promotion, durable-execution engines (pattern donors
only).

## Open before/while ticketing
- **Garden service language (D10): TypeScript vs Python — needs Terra.**
- Reason-code taxonomy v1 confirm (G8.1 list above — topic 13 §6).
- Emergency action set contents per app (H open question; needed for G3.5).
- Which 4-5 apps, named (blocks G6.4 onboarding estimate).
