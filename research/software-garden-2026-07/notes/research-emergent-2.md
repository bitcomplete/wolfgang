# Research: Risk-tiered approval gates and least-privilege authorization for agents operating production systems

Date: 2026-07-29. Researcher: emergent-research agent (web research pass).
Feeds: late-July software-garden release spec; first user is a JUNIOR engineer supervising
agents across 4-5 production apps.

Discovery context this grounds (local files):
- `/Users/terra/Developer/wolfgang/research/software-garden-2026-07/notes/repo-trellis.md` — trellis ships a
  six-layer security model (nono kernel sandbox, credential proxy, SDK tool policy,
  JSONL audit, per-agent tool lists, Sigstore-signed agent instructions) but
  `sandbox_enabled: false` by default and agents run `permission_mode: bypassPermissions`.
- `/Users/terra/Developer/wolfgang/research/software-garden-2026-07/notes/repo-mahdi.md` — mahdi doctrine §2-3:
  "risk tiering by blast radius … QA/approval budget proportional to reversibility" — an
  unvalidated policy skeleton this research validates.
- `/Users/terra/Developer/wolfgang/research/software-garden-2026-07/notes/emergent-topics.md` — Topic 2 framing;
  competitors (Buzz, Efecto) punt on approval gates entirely.

Verification labeling: items marked **[verified]** come from fetched primary pages or
multiple independent sources; **[reported]** items come from search-result summaries or
secondary blogs I did not fully fetch; **[inference]** is my synthesis.

---

## 1. Risk-tiering frameworks for agent actions

The public consensus has converged hard on a small number of tiering axes:
**reversibility, externality (does it leave our systems), and consequence magnitude.**

### OWASP AI Agent Security Cheat Sheet [verified]
Source: https://cheatsheetseries.owasp.org/cheatsheets/AI_Agent_Security_Cheat_Sheet.html

The most directly reusable artifact found. Prescribes:
- Risk levels: **LOW** (read ops, safe queries — auto-approved), **MEDIUM** (writes, API
  calls — review recommended), **HIGH** (financial transfers, deletion, external
  communications), **CRITICAL** (irreversible, security-sensitive).
- For destructive/financial actions: "separate decision-making from execution" and
  **"bind approval to the exact action"** — the approval record must include actor, tool
  name, target resource, normalized parameters, timestamp, and expiry. (This kills
  TOCTOU/approval-reuse attacks; an approval is not a blanket grant.)
- Least privilege: minimum tools per task, per-tool scoping (read-only vs write, specific
  resources), allowlists not wildcards, block `*.env`/`*.key`/`*secret*` file patterns.
- Logging: all agent decisions/tool calls/outcomes; for high-risk actions structured
  metadata — action classification, risk score, authorization outcome, approval
  identifier, execution result, **policy version**.
- Alert on: drift in approval behavior, repeated approval-bypass attempts, elevated
  privilege usage, abnormal tool invocation frequency, spikes in high-risk actions.

### Four-tier action framework (MindStudio) [verified]
Source: https://www.mindstudio.ai/blog/classify-ai-agent-actions-by-risk

| Tier | Name | Governance |
|---|---|---|
| 1 | Read-only | Fully autonomous |
| 2 | Reversible | Autonomous + structured logging + rate limits |
| 3 | External (reaches third parties) | Staging queue for review / confidence thresholds / rate limits |
| 4 | High-risk (irreversible: deletes, money, access-control changes, prod deploys) | Human approval required |

Three classification questions: (1) does it change anything, (2) how easily can incorrect
execution be fixed, (3) does it affect external parties/systems. Quote: "Human approval
for Tier 4 actions isn't a weakness in the system — it's a deliberate design choice."

### Other tiering variants [reported]
- Three-tier READ (auto, silent) / WRITE (auto + log) / DESTRUCTIVE (halt and wait):
  https://dev.to/thedailyagent/how-to-add-human-approval-to-ai-agent-actions-keg
- Blast-radius routing with "compact evidence packs and exception-only review for mature
  low-risk actions": https://cordum.io/blog/approvals-for-autonomous-workflows
- Three gate patterns: pre-execution approval, post-execution review, and
  escalation-trigger (autonomous until a risk signal — sensitive data, irreversibility,
  low confidence): https://www.elementum.ai/blog/human-in-the-loop-agentic-ai

**[inference]** mahdi's blast-radius skeleton is validated: every public framework keys
the human gate on reversibility and externality, exactly as doctrine §2-3 sketches. The
garden can adopt the 4-tier version nearly verbatim and cite OWASP for the approval-record
schema.

---

## 2. Autonomy-level frameworks (who supervises, and how much)

### "Levels of Autonomy for AI Agents" (arXiv 2506.12469, Knight Institute) [verified]
Source: https://arxiv.org/abs/2506.12469 ; https://knightcolumbia.org/content/levels-of-autonomy-for-ai-agents-1

Five levels named by the **user's role**: operator → collaborator → consultant →
**approver** → observer. Key claim: autonomy level is a *deliberate design decision,
separate from capability*. Proposes "AI autonomy certificates" to govern behavior.
**[inference]** The garden's junior is explicitly an **approver** for tier-3 actions and
an **observer** for tiers 1-2 — naming the role clarifies UI design (approval inbox vs
activity feed).

### CSA "Agentic AI Autonomy Levels and Control Framework" [verified]
Source: https://cloudsecurityalliance.org/blog/2026/01/28/levels-of-autonomy ;
PDF: https://labs.cloudsecurityalliance.org/wp-content/uploads/2026/03/agentic-ai-autonomy-levels-control-framework-v2-csa-styled.pdf

Six levels modeled on SAE J3016 driving automation:
- L0 no autonomy (inform only) · L1 assisted (approve every action) · L2 supervised
  (approve **plans/batches**, agent executes within approved scope) · L3 conditional
  (autonomous inside machine-readable boundaries, escalate at boundary) · L4 high
  (exception handling + kill switch, 24/7 monitoring) · L5 full (explicitly "not
  appropriate for enterprise deployment today").
- Controls per level: L2 needs boundary compliance + execution monitoring + rollback;
  L3 needs machine-readable boundary definitions, **technical enforcement** (prevent, not
  detect), escalation workflows; L4 needs anomaly detection, kill switches, full logging.
- **Authorization authority scales with autonomy**: business-owner sign-off for L1,
  manager/security for L2, formal review for L3, executive for L4. "Autonomy alone
  doesn't determine risk" — map autonomy × capability type (money, code exec, physical).

**[inference]** For the July release: runtime-ops agents on production apps should ship at
CSA **L2 (plan-level approval) for mutating work and L3 (bounded autonomy) only for
read/diagnose patrol loops**. L4 is out of scope — the release has no 24/7 anomaly
monitoring story yet.

---

## 3. Change-management precedent: ITIL and SRE

### ITIL 4 change enablement — the three change classes [verified, multiple sources]
Sources: https://www.manageengine.com/products/service-desk/it-change-management/it-change-types.html ;
https://itsm.tools/change-enablement/ ; https://trustedinstitute.com/concept/itil-4-foundation/change-enablement/standard-normal-emergency-changes/

- **Standard change**: low-risk, repeatable, **pre-authorized**, documented procedure,
  often automated — no per-instance approval.
- **Normal change**: individually risk-assessed, scheduled, and authorized (minor vs
  major sub-classes by impact).
- **Emergency change**: expedited assessment/approval to restore service; fast-tracked,
  reviewed after.

**[inference]** ITIL's deepest lesson for the garden is the **standard-change register as
a promotion mechanism**: an action type starts life as a "normal change" (human-approved
every time) and is *promoted* to pre-authorized "standard change" only after a documented
procedure exists and a track record accumulates. This is the principled answer to "how
does the agent earn autonomy" — autonomy is granted per action-type, by humans, based on
evidence, not asserted by the agent. Also note ITIL's emergency class maps to incident
response: the junior needs a defined fast path (act now, review after) or they will
bypass the system during an outage.

### Google SRE: progressive rollout, canary, blast radius [verified, multiple sources]
Sources: https://opensource.com/article/22/6/change-management-site-reliability-engineers ;
https://google.github.io/building-secure-and-reliable-systems/raw/ch07.html ;
https://www.plural.sh/blog/what-is-canary-deployment/

- SRE change management = three tenets: **progressive rollouts, monitoring, safe fast
  rollback**.
- Canarying (SRE Workbook ch. on canarying releases): partial, time-limited deployment
  evaluated against a control; "arguably the best blast-radius-to-cost ratio available";
  staged traffic (5% → 20% → 50% → 100%) with instant rollback.
- Building Secure & Reliable Systems ch.7 covers designing for recovery/rollback as a
  security property.

**[inference]** For agent-initiated deploys, the gate should not be a binary
approve/deny: the *approved unit* is "deploy to canary + auto-rollback on SLO breach,"
which shrinks what the junior must judge. Reversibility is manufactured by the deployment
mechanism, downgrading a tier-4 action toward tier-2.

---

## 4. Least-privilege agent identity and credential brokering

### OWASP LLM06 "Excessive Agency" [verified, multiple sources]
Sources: https://genai.owasp.org (via) https://www.indusface.com/learning/owasp-llm-excessive-agency/ ;
https://auth0.com/blog/mitigate-excessive-agency-ai-agents/

Three root causes: **excessive functionality** (tools beyond task scope), **excessive
permissions** (tools run with broader privilege than needed), **excessive autonomy**
(high-impact actions with no human in the loop). Mitigations: least privilege enforced
*externally* (not by the model), JIT/ephemeral tokens, human-in-the-loop for critical
actions. Key point: guardrails inside the prompt don't count — enforcement must live
outside the agent.

### Credential broker pattern (SANS, "confused deputy") [verified]
Source: https://www.sans.org/blog/your-ai-agent-easily-confused-deputy-why-cloud-security-needs-credential-broker

- Problem: prompt injection lets an attacker *misuse legitimately held credentials*;
  agents concentrate access across many services, so static service-account scopes fail.
- Pattern (CB4A): **agents never hold real credentials.** A Policy Decision Point
  evaluates each credential request; a separate Credential Delivery Point mints
  short-lived (seconds-to-minutes) tokens bound to agent identity (DPoP), scoped **per
  operation, not per service**. Compromising either half alone yields nothing. Canary
  credentials detect compromise. Includes its own tiered-approval framework (auto for
  low-risk, manual for high-risk credential grants).
- Foundation: SPIFFE/SPIRE workload identity + OAuth 2.0 + zero trust.

### Workload identity standards landscape [reported]
- SPIFFE as the assumed standard for agent identity; short-lived X.509 SVIDs,
  auto-rotation, federation: https://www.hashicorp.com/en/blog/spiffe-securing-the-identity-of-agentic-ai-and-non-human-actors ;
  https://www.paloaltonetworks.com/blog/identity-security/ai-agent-security-spiffe-machine-identity/
- **NIST NCCoE concept paper (reported as Feb 2026)** names SPIFFE/SPIRE, OAuth 2.0, and
  zero-trust architecture as candidate standards for AI-agent identity (reported by
  https://www.digitalapplied.com/blog/agent-identity-credentials-non-human-access-2026-playbook —
  not independently verified against nist.gov).
- CSA "Agent Identity Governance Framework" and "Non-Human Identity Governance Vacuum":
  https://labs.cloudsecurityalliance.org/agentic/agentic-identity-governance-framework-v1/ ;
  NHI:human ratios reported at 80-144:1.
- Gap repeatedly flagged: SPIFFE proves *who* an agent is but not *what it may do* —
  authorization policy is the open layer: https://nhimg.org/articles/spiffe-and-ai-agent-identity-expose-the-next-authorization-gap/

### Just-in-time / temporary elevated access (the human analogue) [verified, multiple sources]
Sources: https://goteleport.com/learn/temporary-elevated-access-management/ ;
https://goteleport.com/blog/just-in-time-access-request/ ;
https://docs.aws.amazon.com/singlesignon/latest/userguide/temporary-elevated-access.html

Teleport Access Requests: user (or bot) self-initiates a request for a role elevation;
approver (optionally via Slack/PagerDuty/Jira) approves; short-lived certificate carries
the elevated role until TTL, then reverts. AWS IAM Identity Center ships temporary
elevated access natively. **[inference]** This is the exact shape for garden agents:
default credential = read-only per app; mutating credentials are *requested per task*,
approved by the junior (or auto-approved for standard changes), and expire with the task.

### Trellis parallel [local, verified in discovery notes]
Trellis L2 already implements broker-lite: nono's localhost proxy injects auth headers so
agents never see raw keys (`sandbox_proxy_credentials`, 1Password/Keychain maps). The
public pattern says: keep that, add per-task scoping + TTL.

---

## 5. Sandbox-by-default execution

### OpenAI Codex — the cleanest public two-axis model [verified]
Source: https://developers.openai.com/codex/agent-approvals-security
(redirects to https://learn.chatgpt.com/docs/agent-approvals-security)

Two orthogonal axes: **sandbox mode** (what is technically possible) × **approval
policy** (when to stop and ask):
- Sandbox: `read-only` (default in non-version-controlled dirs) / `workspace-write`
  (default in repos; network blocked by default; `.git` protected) / `danger-full-access`
  (explicitly not for production).
- Approval: `untrusted` (only known-safe reads auto-run) / `on-request` (default —
  escalate on sandbox violation, network access, destructive git ops, side-effectful
  MCP calls) / `never` (CI) / **`auto_review`** (an automated reviewer *agent* screens
  approval requests for exfiltration/credential/destructive risk, **fails closed** on
  parse errors or timeouts).
- Recommended pairings are documented per intent (dev, exploration, CI, delegated
  review).

### Claude Code and the local-agent contrast [reported]
https://paulgp.substack.com/p/permissions-sandboxes-and-autonomous — Claude Code/Gemini
CLI/Copilot run directly on the host with permission *prompts* as the control; Codex
makes OS-level sandboxing the primary control and prompts secondary. Community pattern
writing (https://aipatternbook.com/approval-fatigue, https://www.developersdigest.tech/blog/approval-fatigue-agent-security-bug)
converges on: **enforced boundaries beat prompts**, because prompts decay (see §7).

**[inference]** Trellis's `sandbox_enabled: false` + `bypassPermissions` default is
exactly backwards relative to the strongest public practice (Codex defaults to sandboxed
+ on-request). The release spec should invert the default and treat `danger-full-access`
equivalents as a per-run, logged, expiring exception — never a registry default.

---

## 6. How approval gates are implemented today (CD, ChatOps, agent frameworks)

### CD pipelines [verified, multiple sources]
- **GitHub Environments deployment protection rules**: required reviewers (1-6 people or
  teams), wait timers, branch restrictions, and *custom* protection rules gated by
  third-party checks (Datadog, ServiceNow, Honeycomb):
  https://docs.github.com/en/actions/reference/workflows-and-actions/deployments-and-environments
- **Spinnaker**: built-in Manual Judgment stage. **Argo CD**: no native approval gate —
  teams compose manual sync policies + Git PR review + CI gates:
  https://oneuptime.com/blog/post/2026-02-26-argocd-manual-approval-gates/view ;
  https://www.opsmx.com/blog/spinnaker-vs-argo-cd/
- Regulated-pipeline write-up on approval gates: https://rutagon.com/insights/ci-cd-approval-gates-regulated-systems/

**[inference]** Insight for the garden: in mature CD, the approval is attached to the
**environment/target**, not to the actor — anything crossing into `production` hits the
gate no matter who/what initiated it. Environment-attached gates are robust to "the agent
found another path."

### ChatOps [verified example]
- Policygenius production-access control via Slack: request in channel → on-call manager
  tagged → approve/deny buttons; used for DB rollbacks, credential rotations, feature
  flags: https://medium.com/policygenius-stories/chatops-for-production-access-control-b4feafbe9449
- Same button-approval pattern in runbook automation tooling:
  https://incident.io/blog/runbook-automation-tools-2026-the-complete-guide
- Teleport ships Slack/PagerDuty approval integrations for access requests (see §4).

**[inference]** Approval must arrive where the junior already lives (Slack/dashboard),
with an evidence pack (what, why, blast radius, rollback plan) attached to the button.
Trellis's Telegram/web approval channel is the same shape.

### Agent frameworks [verified, multiple sources]
- **LangGraph `interrupt()` + HITL middleware**: pause at a tool call, persist state via
  checkpointer, resume with a decision — **approve / edit / reject / respond**:
  https://docs.langchain.com/oss/python/langchain/human-in-the-loop ;
  https://www.langchain.com/blog/making-it-easier-to-build-human-in-the-loop-agents-with-interrupt
  The durable-pause detail matters: approvals can take hours; agent state must survive.
- **Vercel AI SDK policy-based tool approvals** with an OPA backend (`@ai-sdk/policy-opa`):
  authorization rules live in Rego, outside the agent, versioned and testable:
  https://ai-sdk.dev/docs/agents/policy-tool-approvals
- **OPA at the tool-calling layer** (ABAC over tool + parameters + context; deterministic
  "regardless of what the LLM thinks"): https://codilime.com/blog/why-use-open-policy-agent-for-your-ai-agents/ ;
  https://www.truefoundry.com/docs/ai-gateway/opa-guardrails (OPA guardrails over MCP tool
  invocations); https://gokhan-gokalp.com/runtime-governance-for-ai-agents-policy-as-code-with-opa/

**[inference]** The policy matrix should be **data, not prose**: a versioned
policy-as-code file evaluated at the tool-call boundary (trellis's `can_use_tool` hook is
the natural enforcement point), so the matrix is testable and its version is stamped into
every audit record (per OWASP §1).

---

## 7. Approval fatigue — why "just ask a human" fails, especially for a junior

Sources: https://aipatternbook.com/approval-fatigue ;
https://www.developersdigest.tech/blog/approval-fatigue-agent-security-bug ;
https://nhimg.org/community/agentic-ai-and-nhis/ai-agent-approvals-and-alert-fatigue-what-teams-are-missing/ ;
https://medium.com/@shreya_edulakanti/approval-fatigue-is-breaking-ai-agents-execution-boundaries-fix-it-6c46c6d512dd [all reported/secondary]

- Documented failure mode: reviewers start careful, then click approve reflexively — "the
  collapse of the oversight assumption." Too many prompts is itself a security bug.
- Converged mitigation: three-stage routing — **DENY rules** (unconditional blocks:
  trellis's Bash blocklist is this) → **ALLOW rules** (auto-approve the safe long tail)
  → **HUMAN gate** only for the residual; plus exception-only review for action types
  with a mature track record (matches ITIL standard-change promotion, §3).
- Codex's `auto_review` (an LLM screens approval requests, fails closed) is the first
  mainstream implementation of a *classifier* stage before the human [verified, §5].

**[inference]** For a junior specifically this is acute: they lack the judgment to
compensate for fatigue, and they will not push back on a confident agent. The matrix must
keep the junior's approval queue *small and meaningful* (target: a handful/day across 4-5
apps), or the safety rail becomes theater.

---

## 8. Signed audit and attestation of agent actions

- Supply-chain stack reusable for agent actions: **in-toto attestation envelope (ITE-6)
  + Sigstore signing + SLSA provenance levels**: https://www.kusari.dev/learning-center/attestation/ ;
  https://aquilax.ai/blog/supply-chain-artifact-signing-slsa [verified concepts, multiple sources]
- Research directly on agent-action attestation [reported, arXiv preprints — treat as
  research-grade, not established practice]:
  - "Verifiability-First Agents" — an Action Attestation Layer emits a signed receipt per
    external call, appended to a hash-chained provenance log: https://arxiv.org/pdf/2512.17259
  - "Proof of Execution: Runtime Verification for Governed AI Agent Actions":
    https://arxiv.org/pdf/2607.05397
  - "Sovereign Execution Broker: Enforcing Certificate-Bound Authority in Agentic Control
    Planes": https://arxiv.org/pdf/2606.20520
  - "Governing Actions, Not Agents": https://arxiv.org/pdf/2606.26298 — governance should
    attach to *actions*, matching the per-message governance idea in Greenwood D8.
- Practitioner pattern: append-only log where each entry carries intent id, action type,
  signed receipt, and a commitment to prior entries (hash chain / Merkle):
  https://zylos.ai/research/2026-04-25-agent-identity-provenance-signed-audit-trails/ [reported]

**[inference]** Trellis L4 (append-only `pool/audit.jsonl`) + L6 (Sigstore-signed agent
instructions, runtime verification via `trust-policy.json`) already cover the two ends —
signed *inputs* and logged *actions*. The missing middle, per the literature, is signing
the action records themselves (hash-chaining audit.jsonl is cheap); full attestation is
post-release. Note: cryptographic attestation of agent actions is **not yet standard
industry practice** — the arXiv work is 2025-2026 preprints; don't over-claim maturity.

---

## 9. Name-collision / scarcity notes

- "Agent approval gates" vendor content (Cordum, MindStudio, Skyrelis, Elementum) is
  young and marketing-adjacent; the tier taxonomies agree with OWASP, which is the
  citable anchor.
- The NIST NCCoE Feb 2026 agent-identity concept paper is cited by several secondary
  sources but was not fetched from nist.gov — verify before quoting in the spec.
- No authoritative public data was found on *quantitative* thresholds (e.g., "N clean
  runs before promotion to standard change") — that number is a judgment call; record it
  as an open question with a starting default rather than presenting it as researched.

---

## 10. Implications for the late-July release (junior operator, 4-5 production apps)

1. **Adopt a 4-tier action matrix as versioned policy-as-code**, enforced at the
   tool-call boundary (trellis `can_use_tool` / OPA-style), not in prompts:
   - **T1 read/diagnose** — auto, silent. (Patrol agents live here; per-agent read-only
     tool lists, trellis L5 style.)
   - **T2 reversible internal writes** (scale a deployment that auto-rolls-back, restart
     a pod, toggle a flag with kill switch) — auto + structured log + rate limit.
   - **T3 external or hard-to-reverse** (customer-visible messages, config affecting
     third parties) — junior approves from an evidence pack (what/why/blast
     radius/rollback), Slack or dashboard button.
   - **T4 irreversible/critical** (data deletion, credential/access changes, schema
     migrations, full prod deploys without canary) — junior may NOT approve alone:
     **two-person rule** — junior + Terra (or on-call senior). CSA's
     "authorization authority scales with autonomy" grounds this; it directly answers
     mahdi's "what a junior may approve alone versus escalate."
2. **Bind approvals to exact actions** (OWASP): approval record = actor, tool, target,
   normalized params, timestamp, expiry, policy version. An approval is not a session
   grant. Durable pause (LangGraph-style checkpointing) so approvals can take hours.
3. **Invert the trellis defaults for anything touching the 4-5 apps**:
   `sandbox_enabled: true`, no `bypassPermissions`; model on Codex's two-axis default
   (workspace-scoped sandbox × on-request approvals); `danger-full-access` equivalent
   only as a logged, expiring, per-run exception.
4. **Manufacture reversibility**: agent deploys go canary + auto-rollback on SLO breach
   (SRE tenets), which demotes deploys from T4 to T2/T3 and shrinks the junior's
   judgment burden. The approved unit is "canary with rollback," never "prod deploy."
5. **Autonomy is earned per action-type via a standard-change register** (ITIL): every
   action type starts at T3 (human-approved); promotion to pre-authorized requires a
   documented procedure + clean track record + explicit human sign-off (by Terra, not the
   junior). Demotion is automatic on any incident. Starting threshold is an open
   question — pick a default (e.g., N=20 clean runs) and record it as unvalidated.
6. **Keep the junior's queue small** (approval fatigue is a documented security bug):
   DENY-list → ALLOW-list → human residual; target a handful of approvals/day. Consider
   a Codex-style fail-closed auto-review classifier as a fast-follow, not for the
   release.
7. **Per-agent identity + brokered credentials**: keep trellis's credential proxy;
   default credential per agent per app is read-only; mutating credentials are minted
   per task with TTL (Teleport/JIT pattern), scoped per operation. SPIFFE/SPIRE is the
   direction but is post-release plumbing; the proxy + TTL scoping is release-sized.
8. **Emergency path**: define the ITIL emergency-change equivalent now — during an
   incident the junior can invoke a break-glass mode that widens T2/T3 (never T4-alone),
   pages Terra, and forces a post-incident review. If this path doesn't exist, the
   junior will bypass the system exactly when it matters most.
9. **Audit now, attest later**: append-only JSONL with the OWASP metadata fields +
   hash-chaining is release-sized; Sigstore-signed action receipts (per the 2026 arXiv
   line and trellis L6 precedent) are the differentiator roadmap, honestly labeled as
   research-grade.
10. **Differentiation claim is supported**: Buzz and Efecto have nothing here, and the
    strongest public practice (OWASP tiering, Codex two-axis defaults, ITIL promotion,
    brokered credentials) is assembleable from parts trellis already half-ships. The
    garden can credibly market "safe-by-default agent operations for non-expert
    operators" if and only if the defaults are flipped on.
