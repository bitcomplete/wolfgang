---
title: "Deep dive: break-glass / emergency access design for the garden's approval gates"
date: 2026-07-29
topic: break-glass and emergency access — SRE practice, gated-automation escape hatches, junior-operator patterns, audit/culture
status: draft
---

Round-2 research pass. Builds on `research-emergent-2.md` §3/§10.8 (ITIL emergency-change
class; "define a break-glass mode that widens T2/T3, never T4-alone, pages Terra, forces
post-incident review — or the junior bypasses the system") and
`reports/hci-for-agents.md` §2.4.7 / must-have #6 (break-glass must exist in the UI,
logged and expiring, before the first real incident). **Nothing below repeats that.**
This pass supplies what those left open: what qualifies as a trigger, exact scope and
mechanics on kploy/k8s/GitHub, time-bounding numbers, the junior-operator adaptation,
and the audit/culture loop — ending in a concrete R1 proposal.

Verification labeling as in the sibling notes: **[verified]** = fetched primary page;
**[reported]** = search-result summaries; **[inference]** = synthesis.

---

## Scored findings

| # | Finding | Score | Why it matters for Aug 16 |
|---|---------|-------|---------------------------|
| B1 | **Google doctrine (BSRS ch. 5)**: breakglass "bypasses your authorization system completely"; valid when the authz system itself fails (large-scale incorrect denials); restricted to the team that owns the operational SLA; **every use closely monitored and reviewed in team meetings; frequent use = the standard path is inadequate and needs redesign**; must be tested regularly | **HIGH** | The canonical framing: break-glass is a *control*, its use frequency is a *gate-quality metric*, and review is a standing ritual — exactly the loop R1 needs. |
| B2 | **Incident-scoped elevation is the modern shape**: incident.io + Apono — declaring an incident triggers auto-approved JIT access based on incident context; access **auto-expires when the incident closes**; every grant/revocation is logged tied to the incident id | **HIGH** | The design answer for the garden: bind break-glass to a declared *incident object on the bus*, not to a credential. Expiry, audit linkage, and review scheduling all fall out of the incident lifecycle. |
| B3 | **Trigger definition from GCP practice**: break-glass qualifies when a P0/P1 incident is active **and the normal approval workflow is unavailable *or too slow***; invocation should "feel deliberate and have a cost" (mandatory rotation + 24h security review as designed friction); quarterly drills with hard metrics (access in <5 min, alert fires in <60 s) | **HIGH** | Gives R1 its two-part trigger test and its drill/metric numbers. "Too slow" legitimizes the garden's core scenario: gate healthy but Terra unreachable during an outage. |
| B4 | **Junior-safe composition** (no published "tiered break-glass for juniors" standard exists — confirmed by search): assemble from (a) minimum-necessary emergency permissions (HIPAA-lineage: scope emergency access to the least data/functionality needed), (b) **pre-approved emergency runbooks** — vetted safe actions (restart, scale, rollback) with blast-radius caps and auto-rollback if post-action verification fails, (c) **invoke-alone-but-alert-immediately** (every activation alerts a second person in <60 s; no pre-approval required) | **HIGH** | This *is* the R1 scope design: the junior's break-glass widens access only to a reversible, pre-vetted action set; seniority-restriction (Google's answer) isn't available when the junior is the only operator, so restriction must be by *scope*, not by *person*. |
| B5 | **Kill switches as the emergency-action floor**: permanent boolean kill-switch flags; flipping one is instant (~200 ms propagation at LaunchDarkly), reversible, and needs no redeploy; can be wired to observability for auto-disable | **HIGH** | The fastest emergency actions should be pre-wired reversible switches (pause-all-agents, freeze-deploys, disable-integration) so most "emergencies" never need widened *access* at all — they need a switch that already exists at T1/T2. |
| B6 | **Anti-pattern is cultural, not technical**: if break-glass is too easy, "just break glass and fix it" becomes the culture and bypasses least privilege entirely; policy must state use is rare, tied to legitimate need, and always reviewed after | **HIGH** | The exact failure the corpus predicts for the junior — the design must make the sanctioned path *cheaper than routing around* but *costlier than the normal gate*. |
| B7 | **GitHub-native break-glass** (kploy's substrate): rulesets have first-class **bypass actor lists** with audit logging; Actions admins can bypass environment protection rules to force pending jobs; practitioner wrapper: notify on-call lead → document in incident channel → post-incident review ticket | MEDIUM | The paved-road bypass already exists on the garden's deploy path; R1 just decides who is on the bypass list (Terra only) and wraps it in the incident ritual. |
| B8 | **k8s emergency access mechanics**: time-bound RBAC — pre-defined minimal ClusterRole, temporary binding to the requester, automated revocation on expiry, k8s audit logs + optional session recording; Kyverno **PolicyException auto-expires** (generated cleanup policy, e.g. 4 h); Gatekeeper exempt-namespaces/`gatekeeper.sh/exempt` are a documented risk surface needing RBAC restriction + quarterly review | MEDIUM | Supplies the platform-layer implementation if R1's emergency kubectl needs to exist at all (proposal: read/restart-only, pre-provisioned, time-bound). Admission-policy exemption is Terra-tier, not junior-tier. |
| B9 | **Emergency accounts as identity practice** (Entra canonical guidance): ≥2 cloud-only accounts tied to no individual, phishing-resistant creds *different* from normal admin, excluded from conditional-access, **alert on every sign-in (threshold >0, severity critical)**, each use triaged as drill / genuine emergency / misuse, **validated at least every 90 days** ("if you don't test it, you have an assumption") | MEDIUM | The garden mostly needs a *mode*, not an account — but the operational triple (alert-on-every-use, scheduled drills, three-way triage of each use) transfers verbatim into R1's review loop. A last-resort cluster credential in a safe is the M2 item this pattern governs. |
| B10 | **AWS layered break-glass reference** (aws-samples): break-glass IAM users at org root assuming per-account roles; three escalation layers (common emergencies → management account → root user as absolute last resort); all break-glass logins/assumptions funneled to one alerting topic | MEDIUM | The transferable idea is *layering*: garden incident-mode (junior) → platform bypass (Terra) → cluster root (safe). Not AWS-specific work for R1. |
| B11 | **CD-pipeline escape hatches**: Spinnaker manual-judgment stages are role-restricted (only listed roles may judge) and support conditional skip; Argo maps judgment to manual sync; emergency deploy = role-gated in-pipeline bypass, still audited | MEDIUM | Validates keeping the emergency path *inside* the pipeline/paved road rather than beside it; with one operator the role-restriction lesson lands on the T4 boundary (junior can never self-judge a T4). |
| B12 | **Blameless-review grounding**: DORA-lineage data — psychological safety correlates with +47% process improvements and +64% near-miss reporting; elite teams 3× more likely to run consistent post-incident reviews; accountability ("who owns preventing recurrence") vs blame ("whose fault") | MEDIUM | Citable grounding for making break-glass review non-punitive: punish use and the junior stops declaring incidents — the exact route-around failure again, one level up. |
| B13 | Healthcare/ePHI break-glass specifics (view-only defaults, local-console limits); quantum-safe / hardware-key vault details | LOW | Background texture; no R1 action. |

---

## 1. SRE/platform practice — what qualifies, who invokes, how it's bounded

### Google: breakglass as a monitored, reviewed, rare control [verified]

[Building Secure and Reliable Systems ch. 5 (Design for Least Privilege)](https://google.github.io/building-secure-and-reliable-systems/raw/ch05.html):
breakglass "provides access to your system in an emergency situation and **bypasses your
authorization system completely**." Key properties:

- **Valid trigger**: the authorization system itself failing — "large-scale incorrect
  denials of access." It is the escape hatch *for the gate*, not around routine friction.
- **Who**: reserved to the team "responsible for the operational SLA"; in the zero-trust
  network case, usable "only from specific locations" (panic rooms with physical access
  controls) — i.e., invocation is narrowed by *role and context*, not made convenient.
- **Audit + culture**: "all uses of a breakglass mechanism should be closely monitored";
  uses are reviewed in team meetings; **frequent use is read as a signal that the
  standard APIs/paths are inadequate and need redesign** — the frequency-as-gate-quality
  metric the garden should adopt verbatim.
- **Testing**: breakglass must be exercised regularly or it will fail when needed.

Same lifecycle from cloud-ops practice ([Grid Dynamics](https://www.griddynamics.com/blog/break-glass-process-cloud-operations) [verified]):
triggers are IdP/MFA/access-control failures blocking legitimate access; the mechanism
must make "minimal or no assumptions about people or technology" and **must not depend on
external approvals that might themselves be down**; access must be time-limited with
expiring credentials; sessions fully logged (up to keystroke recording); the procedure
must have an owning team and recurring drills.

### Concrete trigger + friction design from GCP practice [verified]

[OneUptime GCP break-glass guide](https://oneuptime.com/blog/post/2026-02-17-how-to-implement-emergency-break-glass-access-procedures-for-google-cloud/view):
the qualifying condition is a **P0/P1 incident + normal (PAM) approval workflow
unavailable *or too slow***. Every use triggers immediate alerts to security and
management channels (log-metric → Pub/Sub → Slack/PagerDuty); mandatory review within
**24 h**; credentials rotated after every use ("break-glass should feel deliberate and
have a cost"); **quarterly drills** with tracked metrics: access obtainable in **<5
minutes**, alert fires in **<60 seconds**. GCP-native JIT elevation via
[Privileged Access Manager / temporary elevated access](https://docs.cloud.google.com/iam/docs/temporary-elevated-access) [reported].

### Cloud emergency accounts: the Entra canon [verified]

[Microsoft Entra emergency access accounts](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/security-emergency-access):
≥2 cloud-only accounts on `*.onmicrosoft.com`, tied to no individual, phishing-resistant
creds (FIDO2/cert) *different from* normal admin methods, credentials that cannot expire
or be auto-cleaned, **excluded from the conditional-access policies that would block them
during the exact emergency they exist for**, permanent-active (not eligible) role
assignment. Operations: **alert on every sign-in** (Azure Monitor custom log search,
static threshold > 0, severity 0-critical); a post-mortem team triages *each* use as
(a) planned drill, (b) genuine emergency, (c) misuse; **validate at least every 90 days**
(sign-in works, list of authorized users current, process documented, staff trained).
Community formulation of the failure mode: emergency accounts drift out of compliance —
"if you don't test it, you don't have an emergency control, you have an assumption"
([AdminDroid](https://blog.admindroid.com/best-practices-for-break-glass-accounts-in-microsoft-entra/),
[chanceofsecurity](https://www.chanceofsecurity.com/post/break-glass-accounts-done-right-securing-emergency-access-in-microsoft-entra) [reported]).

AWS's reference shape ([aws-samples/aws-cross-account-break-glass-example](https://github.com/aws-samples/aws-cross-account-break-glass-example),
[letsgodevops](https://letsgodevops.pl/blog/emergency-access-done-right-aws-break-glass-policy-explained),
[windkube](https://blog.windkube.com/aws-break-glass/),
[Eyal Estrin overview](https://eyal-estrin.medium.com/introduction-to-break-glass-in-cloud-environments-01c1117118fe) [reported]):
break-glass IAM users at the org root assuming per-account break-glass roles; **three
escalation layers** (break-glass users for common emergencies → management-account users
→ root as absolute last resort); all break-glass console logins / role assumptions
forwarded to a single SNS topic for unified alerting.

### Kubernetes: bypassing the platform gates [verified/reported]

- **Emergency kubectl** ([hoop.dev](https://hoop.dev/blog/break-glass-kubectl-access-in-kubernetes-fast-secure-and-auditable-emergency-control) [verified]):
  pre-define a minimal ClusterRole; grant via **temporary, time-bound RoleBinding or
  short-lived service-account token** created by pipeline/automation (not hand-edited
  YAML); automated revocation at expiry; k8s audit logs record every command with
  identity + timestamp, optionally plus terminal session recording; every activation gets
  a post-use review aimed at "reducing future need." Teleport ships the same shape as
  product ([break-glass SSH docs](https://goteleport.com/docs/zero-trust-access/deploy-a-cluster/reliability/breakglass-access/) [reported]).
- **Admission-policy exemptions**: Gatekeeper supports exempt namespaces and
  `gatekeeper.sh/exempt`, and the bypass surface (exempt-namespace abuse, webhook DoS,
  ephemeral containers) is a documented attack playbook — mitigations are RBAC-restricting
  who can mark exemptions and **quarterly review of exceptions to prevent erosion**
  ([Aqua on Gatekeeper bypass risks](https://www.aquasec.com/blog/risks-misconfigured-kubernetes-policy-engines-opa-gatekeeper/),
  [decryptiondigest](https://www.decryptiondigest.com/blog/kubernetes-admission-control-opa-gatekeeper) [reported]).
  **Kyverno has the cleanest native mechanism**: `PolicyException` resources that
  **auto-expire** — Kyverno generates a ClusterCleanupPolicy deleting the exception after
  a set window (e.g., 4 h), after which the rule snaps back into force
  ([Kyverno policy](https://kyverno.io/policies/other/expiration-for-policyexceptions/expiration-for-policyexceptions/),
  [CNCF blog](https://www.cncf.io/blog/2023/03/01/temporary-policy-exceptions-in-kubernetes-with-kyverno/),
  [Nirmata](https://nirmata.com/2023/02/14/temporary-policy-exceptions-with-kyverno/) [reported]).

### GitHub: the paved-road bypass [reported]

kploy is GitHub-native, so this is the garden's actual deploy-path break-glass surface:
**rulesets ship a first-class bypass-actor system** (named users/teams/apps) with audit
logging ([GitHub rulesets GA](https://github.blog/news-insights/product-news/github-repository-rules-are-now-generally-available/),
[available rules](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/available-rules-for-rulesets));
Actions admins can **bypass environment protection rules** to force pending deployment
jobs — GitHub's changelog literally calls it break-glass
([changelog 2023-03-01](https://github.blog/changelog/2023-03-01-github-actions-admins-can-now-bypass-environment-protection-rules/)).
Practitioner wrapper for responsible use: contact the on-call lead, document the reason
in the incident channel, open a post-incident review ticket, then grant temporary bypass
([community discussion](https://github.com/orgs/community/discussions/13836),
[rulesets guide](https://cherradix.dev/blog/github-branch-rulesets-guide)).

**[inference — B7, B8]** For the garden: keep the platform-layer bypasses (ruleset bypass
list, admission exemptions, cluster-admin kubectl) **Terra-only**, and give the junior a
*bus-layer* emergency mode instead (§2, §3). The junior-facing break-glass never needs to
puncture the platform; it needs to widen what the *garden's own gate* allows.

## 2. Break-glass in approval-gated automation — incidents as permission scope

### Declaring an incident changes the permission model [verified]

The strongest 2026 pattern, and the one that answers the garden's design question
directly: [incident.io × Apono](https://incident.io/blog/apono-incident-io-integration) —
when an incident is declared, "responders can request access that's **automatically
approved based on the incident context**" per pre-configured security policy; "access
granted during an incident **automatically expires**"; "every access request, approval,
and revocation is logged and **tied back to the incident that triggered it**." The same
integration supports break-glass outside the on-call rotation with logging and optional
secondary approval, and shift-based auto-grants (access appears when your on-call shift
starts, disappears when it ends). Sibling integration:
[incident.io × Opal](https://incident.io/blog/opal-incident-io-integration) [reported].

**[inference — B2]** Three properties to steal: (1) the **incident object is the
authorization scope** — elevation cannot exist without a declared incident, and closing
the incident revokes it; (2) elevation is **policy-shaped in advance** (what widens is
pre-declared per severity, not negotiated at 3 a.m.); (3) the audit story is free —
every gated action carries the incident id. On the Greenwood bus this is one aggregate:
`IncidentDeclared` → fold enters incident mode → `IncidentClosed`/TTL → mode ends. It
composes exactly with pass-2 F1 (gate in the fold): incident mode is *state the decider
consults*, so the widened tiers are enforced by the same mechanism as the normal tiers.

### CD pipelines: role-gated bypass inside the pipeline [reported]

Spinnaker manual-judgment stages restrict *who may judge* to listed roles
([issue #4792](https://github.com/spinnaker/spinnaker/issues/4792),
[Armory policy hook](https://docs.armory.io/plugins/policy-engine/use/packages/spinnaker.execution/stages.before/manualjudgment/))
and support conditional skip paths and judgment-driven rollback
([safe deployments codelab](https://spinnaker.io/docs/guides/tutorials/codelabs/safe-deployments/));
Argo maps judgment to manual sync ([OpsMx comparison](https://www.opsmx.com/blog/spinnaker-vs-argo-cd/)).
The emergency path stays **inside** the pipeline — audited, role-gated — rather than
beside it. PagerDuty/incident.io runbook-automation gate-steps were covered in
`research-emergent-2-deep.md` §6 (F11); not repeated.

### Kill switches: the reverse break-glass [verified/reported]

[LaunchDarkly kill-switch docs](https://launchdarkly.com/docs/home/flags/killswitch) and
[incident-management writeup](https://launchdarkly.com/blog/using-feature-flags-during-incident-management/):
kill switches are **permanent boolean flags** built for instantly disabling risky
functionality — no redeploy, ~200 ms propagation, trivially reversible; they can be wired
to observability metrics for automatic disable
([runtime control blog](https://launchdarkly.com/blog/kill-switches-progressive-rollouts-user-targeting/),
[Upstat](https://upstat.io/blog/feature-flags-kill-switches)).

**[inference — B5]** The garden should hold this up as the *first* emergency lever, ahead
of any access widening: pre-wired, always-available switches — pause-all-agents,
freeze-deploys-for-app-X, disable-patrol-Y — are reversible T1/T2 actions *by
construction*. Most junior emergencies should be resolvable by switch + rollback, making
the break-glass mode's widened surface small and rarely exercised. (The corpus already
demands an agent kill switch; the addition here is: design the *emergency action set*
around switches so break-glass rarely has anything left to do.)

## 3. The junior-specific problem — break-glass without senior judgment

**Direct finding: there is no published "tiered break-glass for junior operators"
standard.** Searches for tiered/graduated break-glass return generic PAM material; the
concept must be assembled [inference, absence-of-evidence from multiple searches]. The
assembly pieces, each individually well-attested:

1. **Scope-limited emergency permissions.** Healthcare break-glass (the oldest regulated
   form) mandates minimum-necessary emergency access — view-only where possible, limited
   to the data/functionality the task needs
   ([Yale HIPAA break-glass procedure](https://hipaa.yale.edu/security/break-glass-procedure-granting-emergency-access-critical-ephi-systems),
   [Satori](https://satoricyber.com/access-control/break-glass-access-control-systems-the-essentials/) [reported]).
   Google restricts break-glass by *who* (the SLA-owning team, panic rooms); a solo
   junior operator makes that unavailable, so the garden must restrict by *what* —
   the widened set is reversible ops only.
2. **Pre-authorized emergency runbooks — the break-glass action is itself a vetted
   script.** Converged AI-ops/runbook practice: a narrow set of pre-approved, low-risk
   remediations (restart a stuck service, scale replicas, roll back a deployment) may
   execute without per-instance approval, but under a **guardrail policy**: blast-radius
   caps (instances touched), approvals required above severity thresholds, and
   **automatic rollback + escalation if post-action verification shows no improvement
   within a defined window**
   ([tianpan.co — giving the on-call agent a runbook](https://tianpan.co/blog/2026-04-12-ai-assisted-incident-response-giving-your-on-call-agent-a-runbook),
   [incident.io runbook automation guide](https://incident.io/blog/runbook-automation-tools-2026-the-complete-guide),
   [Checkly incident automation](https://www.checklyhq.com/docs/learn/incidents/automation-incident-response/),
   [drdroid on-call runbooks](https://drdroid.io/engineering-tools/creating-a-runbook-for-your-on-call-team) [reported]).
   This substitutes *vetting at design time* for *judgment at invocation time* — the
   precise compensator for a junior's missing intuition, and the same move ITIL
   standard-changes make (pass 1 §3).
3. **Paired break-glass: invoke alone, but it alerts a second person immediately.**
   No source requires pre-approval for genuine break-glass (Grid Dynamics explicitly
   warns *against* approval dependencies that may be down); instead the universal pattern
   is **unblocked invocation + instant second-person notification + mandatory post-hoc
   review** — Entra's alert-on-every-sign-in, GCP's <60 s security+management alert, the
   GitHub practitioner wrapper's "notify on-call lead + review ticket"
   ([Entra](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/security-emergency-access),
   [OneUptime](https://oneuptime.com/blog/post/2026-02-17-how-to-implement-emergency-break-glass-access-procedures-for-google-cloud/view),
   [BeyondTrust](https://www.beyondtrust.com/blog/entry/provide-security-privileged-accounts-with-break-glass-process),
   [Lumos](https://www.lumos.com/blog/planning-for-emergency-access) [reported]).
   For the garden: the junior never waits on Terra to act within the emergency set, and
   Terra is always paged the moment the mode starts.
4. **Layered last resorts** (from the AWS three-layer model, B10): the junior's incident
   mode is layer 1; Terra's platform bypasses (ruleset bypass, admission exemption,
   cluster-admin) are layer 2; a sealed cluster-root credential is layer 3 and is never
   the junior's to use alone.

**[inference — B4]** The junior break-glass formula:
**widen tiers, not identity** (same person, same audited path, more of the *reversible*
action set auto-approved) + **vetted actions only** (emergency runbook = the T2-able
subset: restart/rollback/scale/switch; never delete/data-mutation/secrets) + **paired by
page** (invoke alone, Terra paged in <60 s) + **auto-verified** (each emergency action
checks its own post-conditions and rolls back + escalates on no-improvement).

## 4. Audit + culture — non-punitive but always reviewed

- **Review every use, in a standing ritual.** Google reviews breakglass uses in team
  meetings (B1); Entra triages each use three ways — **drill / genuine emergency /
  misuse** (B9); GCP practice mandates review within 24 h (B3). The triage vocabulary is
  worth adopting verbatim: it lets a review conclude "legitimate use, gate was wrong"
  without blaming the invoker.
- **Frequency is a gate-quality metric, not a compliance violation.** Google: frequent
  breakglass = standard paths inadequate → redesign the paths (B1). For the garden:
  every legitimate-but-blocked invocation is *tier-miscalibration evidence* and should
  spawn a recalibration change (promote that action type, adjust a threshold) — the
  break-glass log literally drives the ITIL standard-change promotion loop from pass 1.
- **Non-punitive review is evidence-backed, not sentiment.** DORA-lineage data: high
  psychological safety → +47% process improvements, +64% near-miss reporting; elite
  teams run consistent post-incident reviews 3× more often
  ([SRE School blameless-postmortem guide](https://sreschool.com/blog/blameless-postmortem/),
  [Rootly](https://rootly.com/incident-postmortems/blameless),
  [Hyperping](https://hyperping.com/blog/incident-post-mortem) [reported]). Punishing
  break-glass use teaches the junior to stop declaring incidents — the route-around
  failure recurring one level up. Blame asks "whose fault"; accountability asks "who
  owns preventing recurrence" ([odd.fyi critique](https://odd.fyi/blog/article/incident-post-mortems-that-change-nothing-the-ritual-of-blameless-accountability/)).
- **Anti-pattern 1 — break-glass becomes the default path.** "If break glass access is
  too easy to get, it becomes a shortcut, and 'just break glass and fix it' becomes the
  culture" ([Lumos](https://www.lumos.com/blog/planning-for-emergency-access),
  [Entro glossary](https://entro.security/glossary/break-glass-access/) [reported]).
  Counter-design: invocation is unblocked but *costly afterward* (page + named incident +
  mandatory review + visible metric), per GCP's "deliberate, with a cost" (B3).
- **Anti-pattern 2 — stale emergency machinery.** Untested accounts drift out of
  compliance; creds expire; the person who knew the procedure left. Counters: drill on a
  clock (90 days Entra / quarterly GCP+Gatekeeper-exception review), with the drill
  itself alerting the monitoring channel so alert plumbing is co-tested (B3, B9;
  [ITBlogs failure modes](https://www.itblogs.ca/blog/itblogs-ca-1/break-glass-accounts-in-microsoft-entra-id-failure-modes-detection-and-hardening-38) [reported]).
- **Metrics for R1** [inference from B1/B3/B9]: invocations/month (expect ~0; >1–2/month
  ⇒ tiers miscalibrated — recalibrate, don't reprimand); % invocations reviewed within
  72 h (target 100%); page-fires-within-60 s (drill-verified); time-to-first-emergency-
  action <5 min (drill-verified); % of reviews producing a tier/runbook change (a
  *healthy* number is >0 — reviews that change nothing are ritual).

---

## R1 break-glass proposal (proposal-for-Terra)

**Status: proposal-for-Terra — not decided.** Composes B1–B6 with the existing corpus
(risk tiers, gate-in-the-fold, kill switches, kploy paved road). Ten lines:

1. **Trigger (two-part test):** a user-visible outage or SLO-breach page is active, **and** the normal gate blocks or is too slow (Terra unreachable / approval pending during impact). Judged by the junior; never pre-approved.
2. **Invocation:** one command + one button — `garden incident declare <app> "<one-line reason>"` — emits `IncidentDeclared{severity, reason, invoker}` on the bus. Unblocked, instant, impossible to do silently.
3. **Paired by page:** declaration pages Terra within 60 s (drill-verified) with the NeuBird-style context doc. The junior never waits for Terra to act inside the emergency set; Terra always knows.
4. **Scope — widen tiers, not identity:** while the incident is open, the **pre-vetted emergency action set** for the named app runs at T2 (auto-approved, logged): kploy rollback to last-good tracked image, restart/scale workloads, agent kill switches / deploy freeze, cache/queue flush if runbooked. **Never** widened: data mutation/deletion, secret changes, admission-policy exemptions, anything T4 — those still require Terra (two-person rule holds even mid-incident).
5. **Mechanics:** enforcement is the same fold-precondition gate (pass-2 F1) — incident mode is decider state, so widened tiers use identical machinery, and every emergency action carries the incident id. All mutations stay on the kploy/GitOps paved road; GitHub ruleset-bypass and cluster-admin kubectl remain Terra-only (layer 2); a sealed last-resort cluster credential is layer 3.
6. **Auto-verification:** each emergency action checks post-conditions (SLO/health) and auto-rolls-back + escalates to Terra if no improvement within its runbooked window.
7. **Expiry:** incident mode auto-expires at **60 minutes** or on `IncidentClosed`, whichever is first; one renewal allowed, requiring Terra's ack (a page response, not a form).
8. **Review:** every invocation gets a blameless review within 72 h, triaged **drill / genuine / misuse**; every *genuine-but-gate-was-in-the-way* invocation must produce a tier-recalibration or runbook change. Close-out note feeds the KB.
9. **Metrics:** invocations/month (>1–2 ⇒ recalibrate tiers, never reprimand), page <60 s, first action <5 min, 100% reviewed, % of reviews producing change.
10. **Drills:** one game-day with the junior before Aug 16 (declare → rollback → close → review), then quarterly — the drill also proves the paging path.

**Open questions:** exact contents of the vetted emergency action set per app (needs a
per-app runbook pass); the 60-minute TTL is a labeled default, not researched practice
(pass-2 open question 3 applies); whether severity levels (SEV1/SEV2) should widen
different amounts in R1 or wait for M2 (recommend: single level for R1).

---

## Sources consulted this pass (beyond those linked inline)

[StrongDM break-glass explainer](https://www.strongdm.com/blog/break-glass) ·
[CyberArk break-glass best practices](https://docs.cyberark.com/manage/latest/en/content/sca/dpaforcloud/breakglass.htm) ·
[Cloudanix break-glass procedure](https://www.cloudanix.com/learn/break-glass-procedure-emergency-access-for-critical-resources) ·
[Teleport break-glass SSH](https://goteleport.com/docs/zero-trust-access/deploy-a-cluster/reliability/breakglass-access/) ·
[Delinea break-glass accounts](https://delinea.com/blog/break-glass-accounts) ·
[Aembit break-glass glossary](https://aembit.io/glossary/break-glass-account/) ·
[NHI managing break-glass accounts](https://nhimg.org/community/nhi-best-practices/how-to-manage-break-glass-accounts-securely-a-best-practices-guide/) ·
[Kyverno PolicyException guide (OneUptime)](https://oneuptime.com/blog/post/2026-01-30-kyverno-policy-exceptions/view) ·
[incident.io user roles](https://docs.incident.io/admin/user-permissions) ·
[PagerDuty runbook automation](https://www.pagerduty.com/platform/automation/runbook/) ·
[LaunchDarkly feature flags 101](https://launchdarkly.com/blog/what-are-feature-flags/) ·
[GitHub admin-rule audit (dev.to)](https://dev.to/mattstratton/check-if-you-are-breaking-your-admin-rules-in-your-github-repos-467b) ·
[Quaxel — runbooks to agents](https://medium.com/@Quaxel/runbooks-to-agents-automating-the-boring-80-of-on-call-5b4d763cfe8b)
