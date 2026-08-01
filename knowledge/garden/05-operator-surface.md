---
doc_type: knowledge
topic: Operator surface — cards, runbooks, staleness honesty, patrols, digests
tags: [operator, cards, runbooks, staleness, patrol, digest, cognitive-debt]
created: 2026-08-01
updated: 2026-08-01
status: current — design settled via research + charters; implementation is R1 work (GP6/GP7)
summary: No new UI - compose Slack/pin/bithub/kploy. Live-derived per-app cards, dual-audience runbooks bound to live facts, staleness-honest answers (stamp/warn/refuse), deterministic-only patrol with a >=90% effective-precision gate and one daily digest.
---

# Operator surface

Governing test for everything here (cognitive-debt report): *does this reduce what
the junior must hold in their head, or add to it?* No new UI applications — compose
Slack (approvals, digests), pin/bithub (cards, docs), kploy (deploys).

## Per-app cards (GP6.1)

Status/version/last-deploy **derived live from the log and platform** —
derive-don't-quote; anything recomputable is never hand-written. Watermark-stamped
("as of N events behind head"). Skip-level overview across the 4-5 apps.

## Runbooks (GP6.2)

One per app, **dual-audience**: human diagnostic blocks + copy-paste agent prompts —
one artifact serves both. Steps bind to live system facts (dashboard URL + embedded
query) so drift breaks visibly. `verified` on a procedure means **executed**, not
re-read. Acceptance: junior and agent each complete a drill from the runbook alone.

## Staleness honesty (GP6.3 — fixes the observed mahdi failure)

Authority is **computed from verification metadata, never asserted by persona**:
- Two-date frontmatter everywhere: `updated` (derived from git) vs `verified`
  (human/execution).
- Every rendered answer carries age/provenance stamps; tiered thresholds
  (dbt `warn_after`/`error_after` shape): stamp → warning banner → refuse.
- Deterministic freshness resolution, not LLM judgment. Event-sourced views get
  staleness free: projection watermark = exact data age.
- A 7-week-stale authoritative answer must be impossible *by construction*.

## Patrol (GP7 — charter accepted into R1 scope, DC-13)

Read-only conformance/hygiene checks. Trust rules (Tricorder evidence):
- **Deterministic sources only in R1** — an LLM may phrase the digest, never
  originate, drop, or re-score findings.
- **≥90% rolling effective precision** per (check, app), junior-measured via one-tap
  fixed/not-useful; 2 not-useful strikes in 14 days auto-demotes to shadow.
- Five v1 checks: health probes (tri-state), deploy-vs-repo drift, git hygiene >7d,
  runbook contract (verified-TTL + live links dereference), cert/backup.
- Every finding: evidence link + attached fix. Fingerprint dedupe,
  auto-close-when-fixed, typed exemptions **with required expiry** via PR.
- **One daily per-app digest** (all-clear = one line); the only DM-interrupt class is
  cert<7d / backup-missing / prod-healthz-down. Shadow-first graduation; anything
  unready ships in shadow — a smaller trustworthy patrol beats a complete noisy one.

## Week-one protections (GP9)

First-day path = one app, one runbook, one approval. Approval queue budget:
exceeding a handful/day is a design bug to fix, not a junior problem. Game-day drill
(incident declare → emergency action → review) with the junior before 2026-08-16.

Provenance: `../../research/software-garden-2026-07/r1-spec-inputs.md` §C/I +
reports/; work: `../../research/R1-WORK-BREAKDOWN.md` GP6/GP7/GP9.
