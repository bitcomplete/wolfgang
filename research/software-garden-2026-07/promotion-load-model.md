---
doc_type: model
topic: Promotion/approval load model — 5 small apps in maintenance mode, 2026 rates
tags: [promotion-rate, approval-load, cve, dependency-churn, d17, model]
created: 2026-08-03
updated: 2026-08-03
status: paper model on verified 2026 rates (notes/research-cve-churn-2026.md); validate against the R1 simulated week (risk-assessment test #4)
summary: Expected junior-facing prompts under D17 for maintaining 5 small apps - ~2.5-3.5 one-tap green promotions/day, ~2-3 exceptions/week, ~2-5 same-day KEV patches/month. Flat vs 2025 despite +49% CVE growth, BECAUSE triage is automated - raw volume lands on agents (compute), not the junior (attention). Extrapolation and the three load-bearing policies below.
---

# Promotion/approval load model (2026 rates)

## Verified 2026 inputs (`notes/research-cve-churn-2026.md`)

- CVEs/year: 25,081 (2022) → 28,902 (2023) → 40,009 (2024) → 48,185 (2025) →
  **~66-72k projected 2026** (35,364 in 1H, +49.5% YoY, ~195/day).
- BUT actionable share is falling: ~6% of CVEs ever exploited (lifetime); ~1% of the
  2025 cohort exploited by year-end; 1H-2026 KEV-to-CVE ratio 1.4%; **<9.5% of
  dependency vulns are function-level reachable** (Endor). AI-assisted discovery
  inflates counts without proportional risk (1,061 AI-attributed CVEs in 1H 2026,
  1.3% exploited; Anthropic Glasswing: 23,019 findings → 126 CVEs → 1 exploited).
- GitHub reviewed advisories ~4-5k/yr through 2025; **May 2026 spiked to
  1,560/month (~5×) and intake now exceeds GitHub's review capacity** — upstream
  review can no longer be the sole filter.
- Dep-PR volume, small repo: ungrouped ~5-15/wk (rising with advisory volume);
  grouped ~1-4/wk or monthly batches [estimate].
- Urgent (can't-wait) patches: ~0.3-1 per app-month, bursty; median CVE→KEV 80 days
  but 23-29% of KEVs exploited at/before publication → a same-day lane is needed for
  the rare urgent ones, not a faster weekly train.

## Junior-facing load, 5 small apps, maintenance mode (under D13/D17)

| Stream | 2026 estimate | Notes |
|---|---|---|
| One-tap green promotions | **~2.5-3.5/day** (12-18/wk) | Weekly dep train/app + bug fixes + occasional unbatched KEV patch; train *contents* grow with CVE volume, train *count* doesn't |
| Exceptions (non-green) | **~2-3/wk** | Bigger batches ⇒ slightly higher per-train gate-failure odds; assume 15-20% early, falling with calibration |
| Same-day KEV lane | **~2-5/month** | 0.3-1/app-month, bursty; unbatched green cards |
| T4 two-person | ~2-4/month | Unchanged |
| Patrol digest | 1/day (read-only) | Unchanged |

Verdict: still inside the handful/day budget — **≈ flat vs the 2025-rate model
despite +49% raw CVE growth** — because growth lands on the agent side (patch
evaluation, reachability analysis, test runs = compute), not on human attention.

## Extrapolation (2027-2028, if +40-50%/yr continues)

- Raw CVEs ~95-140k/yr; ungrouped dep-PRs 15-40/wk/app — **unsurvivable for
  human-reviewed flows, immaterial for this design**: junior load stays
  ~flat-by-construction; the lines that grow are agent compute (≈ linear in advisory
  volume) and exception quality pressure.
- The design's exposure is therefore **gate precision under bigger batches** (a
  20-package train needs bisection-on-failure so one bad bump doesn't block 19) and
  **triage quality** (see policies below), not prompt count.

## Three load-bearing policies (without these the model breaks)

1. **Deterministic vuln triage in the pipeline** — EPSS/KEV/reachability filtering
   as patrol/train policy (only reachable-or-KEV vulns escalate; the rest ride the
   weekly train). Upstream (GitHub) review capacity is saturated as of May 2026 —
   our own reachability check becomes App-Operations-Contract-worthy.
2. **Grouping policy in the App Operations Contract** — weekly train per app, KEV
   same-day lane excepted. Ungrouped = 100-200 prompts/wk by 2027.
3. **Train bisection on gate failure** — auto-split failing batches so exceptions
   stay item-sized, not batch-sized.

Validate: R1 simulated week (risk-assessment test #4) against these numbers.
