---
title: "Patrol trust: making automated conformance checking trustworthy and low-noise"
date: 2026-07-29
topic: false-positive budgets, prior art in mechanical conformance, LLM-checker precision, and shadow-mode rollout for the garden's patrol agents
status: draft
---

# Patrol trust — round-2 deep research

**Scope:** the release leans on read-only (T1) patrol agents (doc-vs-reality drift, unpushed branches, stale runbooks, health/kploy contract conformance, cost anomalies). The binding constraint: junior trust is fragile — "three bogus patrol findings in week one kills the release" (`notes/research-practitioner-lessons.md` §6.7), and approval/alert fatigue is a documented security bug. This note answers: what makes automated finding-streams trusted, what precision they need, whether LLM checkers can meet it, and how to roll checks out without burning trust.

**Does not repeat:** `reports/standardizing-across-apps.md` (golden paths, scorecard-lite framing, ops-contract recommendation) or `notes/research-emergent-1-deep.md` (freshness contracts, dbt warn/error tiers, Tech Insights facts/checks/scorecard shape, runbook live-fact bindings). Cross-referenced where they meet.

**Verification labels:** [V] fetched primary source; [S] search-result summary, not independently fetched; [inference] my synthesis.

## Scored findings

| # | Finding | Score | Why for Aug-16 |
|---|---------|-------|----------------|
| 1 | Google Tricorder: "effective false positive" = developer took no positive action after seeing it; production checks must stay **<10% effective FPs**; "not useful" button feeds removal; noisy analyzers get disabled | **HIGH** | The only published, operationalized precision gate for a finding-stream. Defines the garden's number: ≥90% effective precision or the check doesn't face the junior. |
| 2 | Tricorder's four admission criteria: understandable to any engineer, actionable with fix attached, <10% eFP, significant quality impact | **HIGH** | A literal admission checklist for R1 patrol checks. "No fix attached → not a check" is the strongest single noise filter. |
| 3 | SAST field data: 35–91% of warnings unactionable; 56% never addressed in project history; Microsoft devs explicitly prefer false negatives over false positives | **HIGH** | Quantifies the death spiral the release must avoid; justifies deliberately under-checking in R1. |
| 4 | SRE alerting doctrine (Ewaschuk/Google): every page urgent, important, actionable, real; err on the side of *removing* noisy alerts; over-alerting is the harder problem | **HIGH** | Direct doctrine for patrol severity classes: almost nothing a patrol finds is page-worthy. |
| 5 | Digest/batching evidence: batching to ~3×/day improves productivity (moderate effect size); a digest of 8 meaningful items beats 8 pings users learned to ignore — trust in the channel itself recovers | **HIGH** | Settles the delivery question: daily digest, not per-finding messages. |
| 6 | Untuned LLM reviewers run 40–80% FP vs. ~3.2% for years-tuned deterministic rules; OSS maintainers describe LLM-generated reports as a "denial-of-service" | **HIGH** | Kills "LLM writes findings" for R1. Deterministic checks only as finding *sources*. |
| 7 | LLM agents as FP *filters* behind deterministic detectors: FP rate 92%→6.3% (OWASP), 95.5% precision (SWE-agent + Claude Sonnet 4); caveat: aggressive filtering suppresses true positives; strongly model-dependent | **HIGH** | The right LLM role exists and is evidenced: triage/suppress behind a deterministic detector, never originate. |
| 8 | Refute-or-Promote: adversarial "kill gates" + cross-model critic eliminated 79–83% of candidate findings pre-report; 10 reviewers unanimously endorsed a *non-existent* vulnerability — only empirical testing killed it | **HIGH** | Verify-by-execution beats model consensus. Post-R1 pattern for any LLM-originated finding: must reproduce or it doesn't ship. |
| 9 | Kyverno's audit→enforce path: ship policies in Audit, watch PolicyReports, graduate to Enforce as confidence grows; "enforce on day one breaks everyone" | **HIGH** | The industry-standard staged rollout, directly copyable as patrol shadow mode. |
| 10 | Shadow-mode discipline: define release gates *before* shadow starts; evaluate on per-segment slices, not global averages (96% overall hiding 70% on one segment = not ready) | **HIGH** | Graduation must be per-check *per-app*, with criteria written down before the patrol first runs. |
| 11 | Scorecard maintainer annotations: in-repo `scorecard.yml`, five typed reasons (test-data, remediated, not-applicable, not-supported, not-detected), shown alongside results, suppress downstream alerts | **HIGH** | The exemption mechanic to copy verbatim: in-repo, versioned, reason-typed, visible — never silent. |
| 12 | GitHub code scanning lifecycle: findings auto-close when the code is fixed; dismissal is typed (false-positive / won't-fix / used-in-tests) + optional comment; dismissed fingerprints don't re-alert | **HIGH** | The finding-lifecycle model: fingerprint dedupe, auto-close-on-pass, typed dismissals that teach the system. |
| 13 | OpenSSF Scorecard: checks are deterministic heuristics scoring 0–10, with **-1 = "could not evaluate" as a distinct state from failure**; every check ships with documented risks + remediation steps | **HIGH** | "Can't check ≠ fail" prevents a whole class of bogus findings (probe timeout ≠ contract violation). Per-check remediation doc is release-sized. |
| 14 | Renovate noise doctrine: grouping, scheduling, automerge; goal "you should see Renovate less and less"; Dependabot converged (cooldown + groups, 2025) after 200-PRs/week revolts | **HIGH** | Convergent evolution across the industry's two biggest bots: group by theme, schedule delivery, silence what needs no human. |
| 15 | Copilot Autofix: findings with a fix attached get fixed 3× faster overall (XSS 7×, SQLi 12×) | MEDIUM | Evidence for finding #2's "fix attached" rule; the garden's version is a copy-paste command or agent prompt per finding. |
| 16 | Soundcheck model: Check/CheckResult/Track/Level/Certification + **dry-run of a check against entities before finalizing** | MEDIUM | Vocabulary + the dry-run habit; the product itself is far too heavy for 4–5 apps. |
| 17 | Tech Insights ships only 3 fact retrievers (entity metadata, ownership, techdocs *presence*); anything like last-updated requires a custom fact retriever; checks = json-rules-engine on cron | MEDIUM | Confirms (per emergent-1-deep §Odds-and-ends) the garden's checks are custom either way — a portal buys nothing at this scale. |
| 18 | QASecClaw fail-open: if the LLM FP-filter errors or returns malformed output, retain the whole batch — never let filter failure silently suppress real findings | MEDIUM | Design rule for whenever the LLM triage layer arrives (post-R1). |
| 19 | Evidence-grounded verification of agent claims cuts false "done" reports 21%→4.2% (Agent Audit / safety-testing evals) | MEDIUM | Same verify-against-environment principle, from the agent-eval side. |
| 20 | Git hygiene tooling norms: dry-run first, auditable report (age, last commit, author, SHA), protected-branch exclusions, report→discuss→delete | MEDIUM | The unpushed-branches/stale-branch patrol has established conventions; report-only in R1 matches them. |
| 21 | LLM-reviewer confirmation bias: PR descriptions/commit metadata bias judgments; redacting metadata recovered 69% of missed detections | LOW | Mostly a false-negative effect; relevant when LLM triage reads finding context, post-R1. |

---

## 1. Prior art in mechanical conformance — what makes these tools trusted

### 1.1 Tricorder: the canonical precision-gated finding platform

Google's Tricorder (the largest deployed program-analysis platform, ~93K findings/day) is the strongest prior art because it *operationalized* trust ([SWE book ch. 20](https://abseil.io/resources/swe-book/html/ch20.html) [V], [ICSE 2015 paper](https://research.google.com/pubs/archive/43322.pdf) [S], [CACM 2018](https://m-cacm.acm.org/magazines/2018/4/226371-lessons-from-building-static-analysis-tools-at-google/fulltext) [S]):

- **Effective false positive:** "An issue is an 'effective false positive' if developers did not take some positive action after seeing the issue." A *technically correct* finding the human shrugs at counts against the check. This is the right definition for the garden: the junior's behavior, not the check's logic, is the ground truth.
- **The 10% budget:** production checks must "produce less than 10% effective false positives" — developers should feel the check points at a real issue ≥90% of the time. Above ~10%, analyzers were "routinely dismissed or disabled by developers."
- **Admission criteria:** understandable to any engineer; "actionable and easy to fix" with guidance attached; <10% eFP; significant quality impact.
- **The feedback loop is structural:** a "Not useful" button files a bug against the check author; checks with high not-useful rates get disabled (the HTML linter was removed for exactly this). The *threat of removal* keeps check authors honest.

**[inference]** Tricorder is the patrol charter in miniature: per-check admission criteria, a numeric effective-precision gate measured from the consumer's clicks, and automatic demotion. The garden should implement the "not useful" button as a one-word reply in the digest thread, and demotion as automatic.

### 1.2 OpenSSF Scorecard: determinism, tri-state results, remediation docs

Scorecard's checks are deterministic heuristics over repository evidence (configs, workflow files, commit history), each scoring 0–10 — and, critically, **-1 means "could not evaluate," which is explicitly *not* a failure** ([scorecard.dev](https://scorecard.dev/) [S], [checks.md](https://github.com/ossf/scorecard/blob/main/docs/checks.md) [S], [Wiz overview](https://www.wiz.io/academy/openssf-scorecard-overview) [S]). Every check ships a documented risk rationale and remediation steps.

**[inference]** The tri-state (pass / fail / inconclusive) is a cheap, high-value discipline for patrols: a probe timeout, an unreachable API, or missing permissions must produce "inconclusive — patrol could not check X" rather than a finding. Conflating "couldn't verify" with "violated" is precisely the class of bogus finding that kills week-one trust.

### 1.3 Scorecard maintainer annotations: exemptions done right

Verified from the repo ([config/README.md](https://github.com/ossf/scorecard/tree/main/config) [V]): maintainers create `scorecard.yml` (or `.scorecard.yml` / `.github/scorecard.yml`) in the repo root:

```yml
annotations:
  - checks: [binary-artifacts]
    reasons:
      - reason: test-data   # the binary files are only used for testing
```

Five predefined reasons: **test-data**, **remediated**, **not-applicable**, **not-supported** (fulfilled a check in a way the tool can't model), **not-detected** (fulfilled in a supported way the tool missed). Annotations display *alongside* results (`--show-annotations`) and suppress downstream code-scanning alerts [S: [scorecard README](https://github.com/ossf/scorecard)].

**[inference]** Copy this shape: per-app exemption file in the app repo, typed reasons, comment required, rendered in the digest as "exempted (reason)" — never silently dropped. Add one field Scorecard lacks: an **expiry date**, so accepted-risk states re-surface instead of fossilizing (congruent with the two-date freshness doctrine in emergent-1-deep).

### 1.4 Soundcheck and Tech Insights: the scorecard products

- Soundcheck's model — Check, CheckResult (pass/fail), Track, Level, Certification; no-code fact collectors (GitHub, Datadog, K8s, PagerDuty…); **Dry Run to test a check against entities without storing results** ([Soundcheck docs](https://backstage.spotify.com/docs/plugins/soundcheck) [S], [v1.30 features](https://backstage.spotify.com/discover/blog/new-soundcheck-features-v1-30-0/) [S]). The dry-run habit is the noteworthy part: even the commercial product assumes you test a check before it counts.
- Tech Insights (Roadie/Backstage OSS) confirmed [V] ([Roadie docs](https://roadie.io/backstage/plugins/tech-insights/)): only three built-in fact retrievers — entity metadata, entity ownership, techdocs *presence* — with checks as json-rules-engine conditions on cron-scheduled facts. Anything like "docs last updated recently" requires a **custom fact retriever**. This closes the question flagged in emergent-1-deep §Verification-status: the gap is real. Even the flagship portal ecosystem gives you presence checks out of the box; freshness/reality checks are custom code everywhere.

**[inference]** Nobody sells the garden's patrol; everyone sells the *frame* (scheduled fact collection → declarative check → per-entity result). At 4–5 apps the frame is a cron job and a findings file; the differentiator is check quality, which is exactly where the precision budget applies.

### 1.5 Renovate/Dependabot: ten years of noise-management evolution

Renovate's own docs narrate the trust arc: automation delight → "this is getting overwhelming" → users ignore the bot ([Noise Reduction docs](https://docs.renovatebot.com/noise-reduction/) [V]). Their answers, all copyable:

- **Grouping** related updates into one PR (default `group:monorepos` presets) — accepting the trade-off that grouped items are harder to debug individually.
- **Scheduling** delivery windows ("leave yourself enough time in a week to actually get the merging done").
- **Automerge** for changes you'd approve anyway — passing updates become "essentially silent."
- Long-term philosophy: **"over time, you should 'see' Renovate less and less."**

Dependabot converged on the same features (grouping, cooldown) years later ([StepSecurity on 2025 enhancements](https://www.stepsecurity.io/blog/announcing-dependabot-configuration-enhancements-cooldown-and-group-support) [S]) after documented 200-PRs/week team revolts ([Dependabot vs Renovate 2026](https://devsecops.ae/dependabot-vs-renovate/) [S], [16 best practices for reducing Dependabot noise](https://nesbitt.io/2026/01/10/16-best-practices-for-reducing-dependabot-noise.html) [S]).

**[inference]** The convergent lesson for patrols: group by theme and app, deliver on a schedule the human chose, and make the "everything is fine" case *silent or one line* — the junior should see the patrol less and less as the fleet conforms, and that is success, not failure of the patrol.

### 1.6 Kyverno: audit mode as institutionalized shadow mode

Kyverno policies carry `failureAction: Audit | Enforce`. Audit allows the resource and writes a PASS/FAIL entry to a PolicyReport CR; Enforce blocks. Documented best practice is explicit: start Audit, build dashboards from PolicyReports, graduate policies to Enforce as confidence grows — enforcing on day one "suddenly fails many teams' deployments" ([Kyverno policy reports](https://kyverno.io/docs/guides/reports/) [S], [OneUptime Kyverno guides](https://oneuptime.com/blog/post/2026-02-02-kyverno-validation-policies/view) [S], [Policy Reporter](https://kyverno.github.io/policy-reporter/) [S]).

### 1.7 Git hygiene checkers

The unpushed-branches/stale-branch niche has settled conventions: dry-run first; auditable report with branch age, last commit, author, SHA; protected-branch exclusions; a report→discuss→(approve)→delete pipeline rather than silent action ([CI-driven cleanup approach](https://medium.com/@shaikmehnaz0/automating-cleanup-of-dormant-git-branches-safe-auditable-ci-driven-approach-fafeeb89efbc) [S], [Remove Stale Branches action](https://github.com/marketplace/actions/remove-stale-branches) [S], [branch aging reports](https://codepulsehq.com/guides/git-branch-aging-report) [S]). Trellis's 127 unpushed commits (`notes/repo-trellis.md`) is precisely this check's target; the convention says R1 reports, never deletes.

---

## 2. False-positive budgets: what precision does a finding-stream need?

### 2.1 The number

Three independent lines converge:

- **Tricorder's operational gate: ≥90% effective precision**, measured by consumer action, with automatic demotion below it (§1.1). The one deployed, published number.
- **SRE paging doctrine: ~100% for interrupts.** "Pages should be urgent, important, actionable, and real"; "every page should be actionable — simply noting 'this paged again' is not an action"; and crucially, **"err on the side of removing noisy alerts — over-monitoring is a harder problem to solve than under-monitoring"** ([Ewaschuk, My Philosophy on Alerting](https://docs.google.com/document/d/199PqyG3UsyXlwieHaqbGiWVa8eMWi8zzAn0YfcApr8Q/edit) [S], [SRE book, Monitoring](https://sre.google/sre-book/monitoring-distributed-systems/) [S], [incident.io best practices](https://incident.io/blog/sre-alerting-best-practices) [S]).
- **The SAST cautionary corpus: streams at 30–70% precision get ignored wholesale.** 35–91% of warnings unactionable across studies ([Understanding Static Code Warnings](https://arxiv.org/pdf/1911.01387) [S], [Pixee: 91% noise](https://www.pixee.ai/blog/sast-false-positives-reduction) [S]); Tencent measured 76–90%+ FPs in industrial deployment ([LLMs reducing FPs in industry](https://arxiv.org/pdf/2601.18844) [S]); **56% of SAST warnings never addressed in project history**, with fix rates strongly correlated to actionability ([ACWRecommender](https://arxiv.org/pdf/2309.09721) [S]); Microsoft developers explicitly accept false negatives to buy lower FP rates ([What Developers Want from Program Analysis](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/07/What-Developers-Want-and-Need-from-Program-Analysis-An-Empirical-Study.pdf) [S]). Outcome of violating the budget: "reduced trust… or blanket suppression of warnings."

**[inference — the garden's budget]** For a junior with fragile trust the gate is Tricorder's, applied twice: **≥90% effective precision per check** (not per patrol run) before a check's findings reach the junior, and **~100% for anything that interrupts** (page/DM). With ~5 apps and daily patrols, three bogus findings in week one ≈ a stream running at 60–80% precision — squarely in the ignored-wholesale band from the SAST data. The budget also implies the *deliberate-false-negative posture*: shipping 5 checks at 95% precision builds the trust that later carries 20 checks; shipping 20 checks at 80% ends the program.

### 2.2 Actionability is the other half of "effective"

- Findings with a machine-attached fix get resolved 3× faster (7× XSS, 12× SQLi) — GitHub's Copilot Autofix telemetry ([Found means fixed](https://github.blog/news-insights/product-news/found-means-fixed-introducing-code-scanning-autofix-powered-by-github-copilot-and-codeql/) [S], [autofix GA changelog](https://github.blog/changelog/2024-08-14-copilot-autofix-for-codeql-code-scanning-alerts-is-now-generally-available/) [S]). Whether the fix is a code patch or a copy-paste command, attaching it converts a *judgment task* (is this real? what do I do?) into a *verification task* (run this, confirm green) — exactly the demotion the junior needs.
- Warning-actionability correlates with fix rate across the SAST literature (§2.1). **[inference]** The garden's finding schema should make the fix field *mandatory*: evidence link + one-command (or one-agent-prompt) remediation, in the uniform alert shape already specified in `standardizing-across-apps.md` §2.4.

### 2.3 Digest vs. individual alerts

- Batching notifications to ~3×/day improved end-of-day productivity with a moderate effect size ([Journal of Occupational Health study](https://academic.oup.com/joh/article/65/1/e12408/7479297) [S]).
- Digest doctrine from notification engineering: consolidation "decouples the volume of events your system generates from the number of interrupts the user experiences"; a digest of 8 meaningful updates outperforms 8 pings the user learned to ignore — **open rates rise because the channel becomes trustworthy again** ([SuprSend on batching/digests](https://www.suprsend.com/post/notification-batching-and-digest) [S], [Courier on notification fatigue](https://www.courier.com/blog/how-to-reduce-notification-fatigue-7-proven-product-strategies-for-saas) [S]).
- Ewaschuk's symmetric rule: alert on symptoms, keep detail on dashboards; anything neither urgent nor actionable must not interrupt (§2.1).

**[inference]** Patrol findings are almost never urgent (they are hygiene/conformance drift, hours-to-weeks timescale). Default delivery: **one daily digest**, grouped by app, with new/resolved/still-open sections. A tiny enumerated *interrupt class* (cert expiring <7d, backup missing, prod health probe failing) may DM — and that class inherits the ~100% budget.

---

## 3. LLM-based checkers: the 2025–26 evidence

### 3.1 LLMs as finding *sources* are nowhere near budget

- Untuned/first-generation LLM reviewers run **40–80% FP** vs. ~3.2% for a years-tuned deterministic ruleset (SonarSource) ([BDIGITAL: diagnosing AI-review false positives](https://tech.bdigitalmedia.io/blog/diagnosing-ai-review-false-positives/) [S]); tuned commercial AI reviewers still self-report 5–15% ([QWE roundup](https://www.qwe.edu.pl/tutorial/ai-code-review-tools-best-practices/) [S]) — vendor numbers, and still at/above the whole budget before the deterministic checks spend any of it.
- OSS maintainers describe the influx of LLM-generated vulnerability reports as "a denial-of-service attack" [S, reported in the arXiv/roundup corpus above] — the ecosystem-scale version of week-one trust death.
- The most instructive single datapoint: in Refute-or-Promote's evaluation, **ten independent LLM reviewers unanimously endorsed a non-existent vulnerability**; only empirical testing killed it ([arXiv 2604.19049](https://arxiv.org/pdf/2604.19049) [V]). Model consensus is not verification; models share blind spots.

### 3.2 Where LLMs demonstrably help: triage behind a deterministic detector

- **Sifting the Noise** ([arXiv 2601.22952](https://arxiv.org/abs/2601.22952) [V]): coding agents (SWE-agent, OpenHands, Aider) used to *filter* SAST alerts cut FP detection from >92% to as low as 6.3% (OWASP benchmark); SWE-agent + Claude Sonnet 4 identified 95.5% of FPs at 95.5% precision on post-cutoff C/C++, vs 36.4% for plain prompting. Two caveats: results are strongly model-dependent, and "aggressive FP reduction can come at the cost of suppressing true vulnerabilities."
- **QASecClaw** ([arXiv 2605.01885](https://arxiv.org/html/2605.01885v1) [S]): same architecture (SAST proposes, LLM filter disposes) with a load-bearing safety rule — **fail open**: if the LLM filter errors or returns malformed output, retain the whole batch; filter failure must never silently suppress real findings.
- **Verify-before-report** generalizes: evidence-grounded verification of agent claims cut false "task done" reports from 21.0% to 4.2% ([Safety Testing LLM Agents at Scale](https://arxiv.org/html/2607.01793v2) [S]); Agent Audit reaches 98.5% precision via layered FP-suppression mechanisms modulating confidence ([arXiv 2603.22853](https://arxiv.org/abs/2603.22853) [S]).
- **Adversarial adjudication** (Refute-or-Promote [V]): stage gates where agents must actively *disprove* candidates, cold-start reviewers to avoid anchoring, and a cross-model critic ("cross-family review can catch correlated blind spots that same-family review misses") killed 79–83% of candidate findings before disclosure — the surviving stream is what gets reported.

### 3.3 What this means for the garden

**[inference]** A clean three-tier architecture falls out, matching the D7 "classifier triage" direction already in the repo's recent commits:

1. **R1: deterministic checks are the only finding sources.** Every R1 patrol check is a boolean/threshold fact with mechanical evidence (HTTP probe, git query, manifest lint, date comparison). LLMs may *phrase* the digest from structured findings but may not add, drop, or re-score findings.
2. **R2: LLM as precision filter, fail-open, on deterministic candidates** — the evidenced 92%→6% pattern — useful when checks with inherent noise arrive (doc-drift semantics, log anomaly triage).
3. **R2+: LLM-originated findings only through a kill-gate:** the checker must reproduce/confirm against the live environment (run the command, fetch the URL, execute the probe) before a finding exists; unverifiable candidates are abstentions ("inconclusive"), not findings; ideally a second model family adjudicates before junior-facing surfacing.

---

## 4. Rollout: shadow mode, graduation, finding lifecycle

### 4.1 Shadow/audit mode first — with gates defined up front

- Kyverno's audit→enforce path (§1.6) is the policy-world template; ML shadow-mode practice adds two disciplines: **define graduation gates before shadow starts** (agreement rate, error classes, stability), and **evaluate per segment, not on the global average** — "a 96% overall rate that hides a 70% rate on one segment is a regression on that segment" ([Brightlume shadow-mode rollouts](https://brightlume.ai/blog/shadow-mode-rollouts-ai-agents-pilot-production) [S], [Samiullah, shadow-mode deployment](https://christophergs.com/machine%20learning/2019/03/30/deploying-machine-learning-applications-in-shadow-mode/) [S]).
- **[inference]** For patrols "segment" = per-check × per-app: a health-probe check can be flawless on four apps and chronically wrong on the fifth (flaky ingress, nonstandard port). Graduation is per (check, app) pair; one pair's noise must not hold back — or contaminate — the rest.

### 4.2 Graduation criteria

**[inference, synthesized from §1.1, §1.6, §4.1]** Concrete, small-fleet-sized gates:

- **Shadow phase:** check runs on full cadence, findings visible to Terra only. Exit requires *both* a minimum duration (7 consecutive days without a false finding) *and* a minimum evaluated volume (every distinct finding fingerprint the check produced has been adjudicated true/false — at garden scale that's usually <10 fingerprints). Zero-finding shadow weeks also need a **seeded fault test**: deliberately break the invariant once and confirm the check fires — a check that can't catch a planted violation isn't validated, merely quiet.
- **Live phase, continuous:** Tricorder rule — effective precision ≥90% rolling, measured by the junior's per-finding feedback (fixed / not-useful). **Two not-useful strikes in a rolling 14 days auto-demotes the (check, app) pair to shadow** and notifies Terra. Demotion is automatic; re-graduation is manual.

### 4.3 Finding lifecycle

Model on GitHub code scanning ([Resolving code scanning alerts](https://docs.github.com/en/code-security/code-scanning/managing-code-scanning-alerts/resolving-code-scanning-alerts) [V-adjacent, docs read via search]) plus Scorecard annotations (§1.3):

- **Identity:** fingerprint = (app, check-id, resource). One open finding per fingerprint — repeat patrol runs update `last_seen`, never re-notify. (The dedupe rule that prevents "same stale runbook, 7 days running" from being 7 findings.)
- **Auto-close:** fingerprint absent from a successful patrol run → finding closes automatically, reported in the digest's "resolved" section — free positive reinforcement and the visible proof the stream is live, not append-only.
- **States:** open → fixed (auto) | dismissed(typed reason + comment) | exempted (in-repo file, typed reason, **expiry date**) | inconclusive (couldn't check ≠ fail, §1.2). Dismissed fingerprints don't re-alert; expired exemptions reopen.
- **Snooze** = exemption with a short expiry; no separate mechanism needed.

---

## 5. R1 Patrol Charter — proposal for Terra

**Status: proposal, not decision.** Everything below is sized for Aug-16 and the existing T1 (read-only) tier; it operationalizes the "conformance patrol, scorecard-lite" must-have from `standardizing-across-apps.md` §4.6.

### Principles (the contract every check signs)

1. **Deterministic finding sources only in R1.** A check is a mechanical fact with evidence; LLMs may phrase the digest but never originate, drop, or re-score findings. (Findings #6, #7, #8.)
2. **Tri-state results:** pass / fail / inconclusive. Probe failures, timeouts, missing permissions are *inconclusive*, reported as patrol health, never as app findings. (Finding #13.)
3. **Every finding carries:** evidence link (command output, URL, commit SHA), one attached fix (copy-paste command or agent prompt), check-id, first/last-seen. **No fix authored → check not admitted.** (Findings #2, #15.)
4. **Effective precision is the metric,** judged by the junior's action, not the check's logic. (Finding #1.)

### R1 checks (5, all deterministic, all reading surfaces the ops contract already standardizes)

| # | Check | Mechanism | Evidence |
|---|-------|-----------|----------|
| 1 | **Health contract** | `/healthz`, `/readyz`, `/metrics` respond 200 per app (retry ×3 before fail) | probe transcript |
| 2 | **Deploy/repo drift** | kploy tracked image == image built from latest main; no dirty worktree on operator checkouts | image tag + SHA diff |
| 3 | **Git hygiene** | branches with unpushed commits > 7d; report-only, protected branches excluded | branch, age, author, SHA |
| 4 | **Runbook contract** | runbook exists at standard path; `verified` within warn/error thresholds (dbt-shape, per emergent-1-deep §2.1); embedded live-fact links dereference 200 | file path, dates, dead links |
| 5 | **Cert + backup** | ingress cert expiry > 14d; declared `bc-prod.yaml` backups ran within cadence | expiry date, last-backup timestamp |

**Deliberately deferred:** cost-anomaly (needs baseline history — same reason anomaly-learned freshness scored LOW in emergent-1-deep), semantic doc-vs-reality drift (needs the R2 LLM triage tier), kploy overlay lint (add as check #6 once 1–5 graduate).

### Precision gate before junior-facing

- Shadow first: findings to Terra only. Graduate per **(check, app)** pair after 7 clean consecutive days + all fingerprints adjudicated + one seeded-fault firing test.
- Live: ≥90% rolling effective precision per check; junior gets a one-tap **fixed / not-useful** response per finding; **2 not-useful strikes / 14 days → auto-demote pair to shadow**, Terra notified. Demotion automatic, re-graduation manual.

### Digest cadence

- **One daily digest** (morning, per-app grouped): new / resolved-since-yesterday / still-open (with age). All-clear day = one line, not silence.
- **Interrupt class (DM-worthy), enumerated and closed:** cert <7d, backup missing, prod healthz failing. Nothing else interrupts; this class carries a ~100% precision expectation.
- Weekly one-screen trend rollup (open findings by app, checks in shadow, precision per check) for Terra.

### Exemption mechanics

- Per-app `garden.yml` (Scorecard-annotation shape): check-id + typed reason (`not-applicable | remediated | accepted-risk | not-supported`) + free-text comment + **required expiry date**.
- Exemptions render in the digest as "exempted (reason, expires …)" — visible, never silent. Expiry reopens the finding.
- R1 approval flow: junior proposes (PR to the app repo), Terra approves — the exemption file is itself in-repo, versioned, and auditable.

### Success criteria for the patrol itself

Zero not-useful strikes in the junior's first 14 days; ≥1 auto-closed finding in week one (the junior must *see* the loop close); every live check has survived a seeded fault. If check #anything can't meet the bar by Aug-16, it ships in shadow — a smaller trustworthy patrol beats a complete noisy one.

---

## 6. Sources not already linked inline

- [Soundcheck plugin page](https://backstage.spotify.com/partners/spotify/plugin/soundcheck/) — check/track/level model [S]
- [ZeroFalse: improving SAST precision with LLMs](https://arxiv.org/html/2510.02534) — additional LLM-filter datapoint [S]
- [Kyverno audit-mode discussion #716](https://github.com/kyverno/policies/discussions/716) — audit/enforce semantics corner cases [S]
- [Multi-model AI code review convergence (Zylos)](https://zylos.ai/research/2026-03-01-multi-model-ai-code-review-convergence/) — multi-pass/multi-model review practice, vendor-research tier [S]
- [Confirmation bias in LLM-assisted security review](https://arxiv.org/html/2603.18740v1) — metadata-redaction effect [S]
- [Adversarial robustness of LLM reviewers](https://arxiv.org/html/2602.16741v1) — comment injection largely ineffective (n=14,012) [S]

**Verification status:** Tricorder ch.20, Renovate noise docs, Sifting-the-Noise abstract, Refute-or-Promote abstract, Roadie Tech Insights docs, Scorecard config/README fetched directly [V]. SRE/Ewaschuk, Kyverno, digest-batching, SAST-FP corpus, shadow-mode, code-scanning lifecycle from search summaries of reputable sources [S] — individually plausible, key numbers (56% never addressed, 35–91% unactionable, batching study) not independently re-derived.
