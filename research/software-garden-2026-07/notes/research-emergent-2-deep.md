---
title: "Deep dive: policy-as-code, credential brokering, approval mechanics, and bus-side gate enforcement for agent operations"
date: 2026-07-29
topic: risk-tiered approval gates and least-privilege authorization (pass 2 — deeper/wider)
status: draft
---

Second research pass. Builds on `research-emergent-2.md` (tiering frameworks, autonomy
levels, ITIL/SRE precedent, approval fatigue, Codex two-axis model, audit/attestation) —
**nothing below repeats that**. This pass goes deep on: concrete policy-engine choices
(Cedar vs OPA vs Kyverno, and who ships what), credential brokering mechanics
(Vault/OBO/workload identity), the maker-checker approval state machine (the exact
mechanics pass 1 lacked), enforcement in an event-sourced message path (approval as an
event the fold requires), and 2025-2026 platform developments pass 1 missed (Bedrock
AgentCore Policy, MCP 2026-07-28 spec, Claude Code hooks, OpenAI AgentKit/Connector
Registry, Microsoft Agent Governance Toolkit).

Verification labeling as in pass 1: **[verified]** = fetched primary page; **[reported]**
= search-result summaries; **[inference]** = synthesis.

---

## Scored findings

| # | Finding | Score | Why |
|---|---------|-------|-----|
| F1 | **Gate in the fold**: model approvals as first-class events (`ActionProposed → ApprovalGranted/Denied → ActionExecuted`); the decider rejects any Execute command whose fold state lacks a matching, unexpired, parameter-bound approval | **HIGH** | This is *the* Greenwood-native enforcement design: on an event-sourced bus the gate lives in the message path and cannot be bypassed by any agent route. Greenfield — costs little now, unretrofittable later. |
| F2 | **Maker-checker state machine + invariants** (centralized approval-request aggregate; PENDING/APPROVED/REJECTED/CANCELLED/PROCESSING/FAILED/EXPIRED; self-approval blocked; entity-version-at-submission so edits invalidate approvals; idempotent execution log; expiry + escalation) | **HIGH** | Gives the release spec the exact schema and invariant list for the approval aggregate — including the failure modes (FAILED ≠ APPROVED, races, stale approvals) a home-rolled design would miss. |
| F3 | **Cedar as the policy engine** for tool-call authorization: default-deny, conditions over LLM-generated tool *parameters*, embeddable (no server), formally analyzable; industry converged here (Bedrock AgentCore Policy GA Mar 2026; Cedar now CNCF) | **HIGH** | Settles "how is the tier matrix represented and evaluated": an off-the-shelf, testable, default-deny engine at the bus's tool-call boundary instead of a home-rolled matrix interpreter. Release-sized (in-process library). |
| F4 | **Enforcement point that survives bypass**: Claude Code PreToolUse hooks veto tool calls even under `bypassPermissions`/`--dangerously-skip-permissions`; deny rules cannot be overridden by any settings level | **HIGH** | Directly fixes the trellis `bypassPermissions` hole flagged in pass 1: the agent-runtime hook calls the bus/Cedar policy check, and bypass mode can't disable it. Concrete, shipped, wire-up-now. |
| F5 | **Paved-road enforcement on bc-prod**: agent changes go through the same kploy/GitOps/admission-controller (Kyverno/OPA) gates as human changes — agents get git-commit provenance and platform policy for free; never raw kubectl credentials | **HIGH** | The 4-5 apps already live on kploy with sealed secrets and CI gates. "Agents act only through the paved road" reuses existing enforcement instead of building parallel guardrails before Aug 16. |
| F6 | **Originating-human authority rides the message envelope** (integrity-protected user context: who initiated, role, auth strength), verified at each hop so delegation never exceeds the initiating human's authority (OWASP ASI03; AWS three-layer Cedar model) | **HIGH** | An envelope-schema decision for the D8 messaging primitive — near-free in a greenfield bus, notoriously hard to retrofit. Junior-operator context makes "agent can't exceed the junior's authority" a core safety property. |
| F7 | Vault-validated credential brokering: OAuth OBO token exchange → Vault JWT auth → dynamic, time-bound per-task credentials; correlation-ID threading for end-to-end attribution | MEDIUM | Confirms and details pass-1 §4 direction; running Vault (Enterprise features) is not release-sized for a tiny company. M2: adopt the OBO + dynamic-secrets shape; release keeps proxy + TTL. |
| F8 | MCP 2026-07-28 spec: stateless protocol; elicitation replaced by `InputRequiredResult` + opaque `requestState` (multi-round-trip); URL-mode elicitation for out-of-band consent; OAuth hardening via RFC 8707/8693 | MEDIUM | Protocol-level "pause for human input" pattern the bus's approval flow should stay compatible with; matters as garden tools standardize on MCP (M2+). Spec explicitly leaves JIT consent/RBAC/audit to the runtime — i.e., to the garden. |
| F9 | Mechanized earned autonomy: Microsoft Agent Governance Toolkit's 0-1000 trust score that decays on violations and recovers with compliance; gates output allow/deny/require-approval | MEDIUM | Automates pass-1's ITIL standard-change promotion. Release keeps promotion human-decided (Terra); a numeric trust signal per action-type is a good M2+ mechanism. |
| F10 | M-of-N / threshold-routed approval policies (impact-conditional approval chains, e.g. >X ⇒ 2 levels), NIST 800-53 AC-3(2)/AU-9(5) dual-authorization controls as citable grounding | MEDIUM | Release needs only the fixed two-person rule for T4 (junior + Terra); conditional M-of-N chains and compliance mapping are M2+ when more humans exist. |
| F11 | incident.io "Intelligent Runbook Execution": approval gates embedded as runbook steps, approver routing driven by a live service catalog (alert→service→team→on-call), audit auto-generated | MEDIUM | With one operator, routing is trivial; the runbook-step-with-gate shape is the M2 pattern for agent-run remediations once more approvers exist. |
| F12 | OpenAI AgentKit Connector Registry: org-wide governance object over which tools/connectors exist, who approved them, who can call them | MEDIUM | Validates a garden "tool registry with approval provenance" (which tool, admitted by whom, on what date, at what tier) — M2 shape; release has a hand-curated tool list. |
| F13 | Full three-layer Cedar evaluation (agent→tool, agent→agent delegation with hop limits, originating-user layer) with HMAC-signed user context and per-hop OBO token exchange | MEDIUM | The envelope field is F6/HIGH; the complete three-evaluator apparatus with token exchange per hop is M2+ — release has few agents and shallow delegation. |
| F14 | Tool-definition supply-chain scanning (hidden instructions, typosquatting in MCP tool descriptions) and response-content validation checkpoints | LOW | Garden authors its own tools for the release; third-party tool ingestion is a later problem. |
| F15 | Quantum-safe agent identity (ML-DSA-65), SPIFFE-compatible crypto identity per agent; SL5-grade two-party control for AI weights | LOW | Interesting horizon material; wildly oversized for 4-5 apps and one junior operator. |
| F16 | Stateless-MCP scalability rationale (any server instance answers any request; kills session-hijack class) | LOW | Greenwood is Kafka-native; the protocol-scaling argument doesn't change the release design. |

---

## 1. Policy-as-code engines for agent tool calls — the 2026 landscape settled

### Bedrock AgentCore Policy: Cedar at the gateway [verified]

Announced at re:Invent, **GA March 3, 2026**. The clearest signal that "policy engine at
the tool-call boundary" is now shipped mainstream practice, not a pattern essay.

From the AWS "why Cedar" post (May 20, 2026,
https://aws.amazon.com/blogs/security/why-policy-in-amazon-bedrock-agentcore-chose-cedar-for-securing-agentic-workflows/):

- Enforcement sits in the **gateway between agents and tools** — "since LLMs cannot be
  trusted to enforce their own constraints, authorization must occur outside the agent."
  **Default-deny**: everything blocked unless a policy permits.
- Cedar was chosen over Rego/general-purpose code for three properties: **analyzability**
  (policies encode as formulas; automated reasoning finds contradictions, overly
  permissive rules, and permissiveness diffs between policy versions), **readability**
  ("structured natural language" auditors can read), and **O(n) evaluation** (no loops,
  no state).
- Policies condition on **untrusted LLM-generated tool parameters**, not just tool names:

  ```cedar
  permit (principal is AgentCore::OAuthUser,
          action == AgentCore::Action::"ApplyBulkDiscount", resource)
  when { principal.getTag("customer_tier") == "Platinum"
         && context.input.orderQuantity >= 50 }
  unless { context.input.productTypes.containsAny(["limited_edition", "seasonal_specials"]) };
  ```

- Explicit position on HITL: human-per-decision is the fallback, not the primary control
  — "human oversight at the policy definition level rather than per-decision." This is
  the approval-fatigue argument (pass 1 §7) from the enforcement side.

Secondary implementation guides: https://hidekazu-konishi.com/entry/amazon_bedrock_agentcore_policy_implementation_guide.html ;
https://clawaws.com/blog/agentcore-policy-review/ ;
https://shinyaz.com/en/blog/2026/03/15/bedrock-agentcore-policy-cedar-authorization [reported].
Cedar joined the CNCF [reported, multiple sources].

### The Cedar/OPA/Kyverno division of labor [reported, converging sources]

- **Kyverno / admission controllers** gate *what gets deployed* into Kubernetes (YAML
  policies, intercepts API-server requests pre-etcd):
  https://devopsboys.com/blog/kyverno-kubernetes-policy-engine-complete-guide-2026
- **OPA or Cedar at the gateway** gates *what running things may do* per request/tool
  call, sub-millisecond per evaluation:
  https://www.agiusalexandre.com/blog/2026-05-16-mcp-gateway-policy-enforcement-rbac-agent-tools/ ;
  https://developers.redhat.com/articles/2025/12/12/advanced-authentication-authorization-mcp-gateway ;
  https://www.permit.io/mcp-gateway
- The AI-SRE framing that matters most for the garden: "a remediation passes through the
  same OPA or Kyverno admission gates as any human change, so a policy violation is
  refused whoever proposed it": https://edixos.com/en/blog/ai-sre-agents-autonomous-operations/ ;
  guidance to route agent changes "through GitOps so every action is a reviewable,
  revertible commit": https://www.fairwinds.com/blog/2026-kubernetes-playbook-ai-self-healing-clusters-growth ;
  https://kaden-projects.com/blog/securing-ai-agents-infrastructure-layer/

**[inference — F3, F5]** Two spec-shaping conclusions:

1. **Don't invent a policy DSL for the tier matrix.** Represent T1-T4 rules as Cedar
   policies evaluated in-process at the bus's tool-call boundary. Cedar is an embeddable
   library (Rust core with bindings) — no policy server to operate, which fits a tiny
   company; default-deny matches the desired posture; conditions over normalized tool
   parameters implement OWASP's "bind approval to the exact action" mechanically; and
   the policy file is diffable/testable/versioned, satisfying pass-1's "policy version
   stamped into every audit record."
2. **On bc-prod, reuse the paved road.** Agents should operate the 4-5 apps only through
   kploy + git (tracked images, kustomize overlays, sealed secrets), never via a raw
   kubectl credential. Then every agent mutation is a commit (provenance for free) and
   existing platform gates apply identically to agents and humans. The garden's
   *additional* gates sit in the bus; the platform's gates come free.

## 2. Approval mechanics: the maker-checker state machine [verified]

Source: https://www.opcito.com/blogs/maker-checker-implementation-guide-for-secure-fintech-systems
— fintech dual-control ("four-eyes") implementation guide. This supplies the concrete
mechanics pass 1's OWASP-level guidance ("bind approval to the exact action") left
abstract. Directly transplantable to the approval aggregate on the bus:

- **Centralized approval-request record**, not per-entity approval columns:
  `{id, operation_type, action, request_payload (normalized params), status, maker_id,
  checker_id, created/reviewed/executed timestamps, entity_version_at_submission}`.
- **State machine**: `PENDING → APPROVED → PROCESSING → (executed) | FAILED`, with
  `REJECTED / CANCELLED / EXPIRED` as terminal states. Two states home-rolled designs
  miss: **FAILED** (approval ≠ successful execution — track the gap) and **CANCELLED**
  (the maker/agent must be able to withdraw a pending request).
- **Self-approval blocked in code**: `maker_id == checker_id ⇒ deny`, tied to individual
  identities. For the garden: *the proposing agent can never be the approving principal*,
  and for T4 the junior cannot approve what the junior initiated — grounding for the
  two-person rule. NIST 800-53 names these as formal controls: AC-3(2) dual
  authorization for privileged commands (https://csf.tools/reference/nist-sp-800-53/r4/ac/ac-3/ac-3-2/)
  and AU-9(5) for audit-record protection (https://csf.tools/reference/nist-sp-800-53/r4/au/au-9/au-9-5/);
  overview: https://en.wikipedia.org/wiki/Two-person_rule
- **Edit invalidates approval**: capture `entity_version_at_submission`; at execution,
  verify the target hasn't changed since the checker looked at it — otherwise fail with a
  conflict. Kills TOCTOU on the *target* side (pass 1's OWASP item killed it on the
  *action* side). Store a `snapshot_before` so the reviewer sees a diff, not a payload.
- **Race safety**: optimistic locking so two concurrent approvals can't both fire;
  compare-and-swap on `status = 'PENDING'`.
- **Idempotent execution**: an execution log with a unique constraint per approval-request
  id; "log the execution BEFORE marking as approved" so crash-retries can't double-execute.
- **Expiry + escalation SLA**: 24h reminder → 48h escalate → 72h auto-expire; expired
  requests never execute.
- **Pitfalls list**: over-applying dual control to low-risk ops (= approval fatigue),
  single-checker bottleneck, ignoring FAILED, missing cancel flow, no notifications.
- Threshold-routed chains for M-of-N: impact-conditional levels
  (`amount > 10000 ⇒ 2 approvers; > 100000 ⇒ 3`), `PENDING_L1 → PENDING_L2 → …`.

The guide explicitly does **not** cover automated actors as makers — the garden extending
maker-checker to agent-makers is genuinely novel ground (differentiation, and a place to
be careful).

**[inference — F2]** Adopt this wholesale as the approval aggregate's contract. Every
item maps 1:1 onto events: `ActionProposed` (maker=agent, payload, target version),
`ApprovalGranted` (checker, expiry), `ProposalCancelled/Rejected/Expired`,
`ExecutionStarted/Succeeded/Failed`. The pitfalls become test cases.

## 3. Enforcing gates in the event-sourced message path

### The pattern: approval is an event the fold requires [inference, grounded]

Grounding: the functional event-sourcing **Decider** pattern —
`decide: (command, state) → events` / `evolve: (state, event) → state` —
https://thinkbeforecoding.com/post/2021/12/17/functional-event-sourcing-decider ;
https://delta-base.com/docs/concepts/functional-event-sourcing-decider/ ;
https://dev.to/jakub_zalas/functional-event-sourcing-1ea5. Saga/orchestration literature
adds that human approval is a long-running wait needing durable state, not a blocking
call: https://ibm-cloud-architecture.github.io/refarch-eda/patterns/saga/ ;
https://floriancourouge.com/en/blog/event-driven-architecture-kafka

Composed for Greenwood:

1. An agent cannot emit `ExecuteAction` directly for gated tiers. It emits
   `ActionProposed{action_type, normalized_params, target, target_version, evidence_pack,
   policy_version}`.
2. The Cedar check runs *in the decider* for the action stream: T1/T2 ⇒ `decide` emits
   `ActionAutoApproved` (policy version stamped) and proceeds; T3/T4 ⇒ `decide` emits
   `ApprovalRequested` and the fold enters PENDING.
3. `ExecuteAction` is **only accepted when the folded state contains a matching,
   unexpired `ApprovalGranted`** whose params/target-version hash equals the proposal's.
   No approval event in the stream ⇒ the command is rejected deterministically, no matter
   which agent, path, or retry produced it. The gate is a *precondition in the fold*, not
   middleware that a new code path can route around.
4. Approvals arrive as ordinary bus events from the human-facing surface (Slack button,
   dashboard) — so the audit trail is the topic itself: proposal, decision, approver
   identity, execution, and outcome are one causally-ordered sequence. Pass 1's
   hash-chained audit JSONL becomes redundant with Kafka's log + (later) signed events.

Three properties fall out for free that bolt-on approval systems must build:
**durability** (a pending approval survives restarts because it *is* state),
**bind-to-exact-action** (the approval event carries the params hash), and
**non-bypassability in the message path** (there is no execute without the event).

### Independent validation of the shape [verified/reported]

- **Temporal HITL**: workflows pause on a Signal "for hours, days or indefinitely" at
  zero compute; approval decisions are durably stored; resume is exact —
  https://docs.temporal.io/ai-cookbook/human-in-the-loop-python ;
  https://learn.temporal.io/tutorials/ai/building-durable-ai-applications/human-in-the-loop/ ;
  https://temporal.io/code-exchange/durable-agentic-harness-crash-safe-autonomous-ai-agents-with-human-in-the-loop-approval
  Temporal is the durable-execution version of the same idea; the event-sourced bus gets
  it natively.
- **The anti-pattern is documented**: implementing HITL as a blocking HTTP call fails in
  production within days (gateway timeouts at 29s, serverless limits); a correct system
  needs a pending-decision store with `pending/approved/rejected/expired` states —
  https://matheuspalma.com/blog/human-in-the-loop-llm-tool-approval-production ;
  https://truto.one/blog/implementing-human-in-the-loop-approval-workflows-for-consequential-saas-api-actions/ ;
  https://www.nnode.ai/blog/2026-02-05-human-in-the-loop-approval-gates [reported]
- **MCP 2026-07-28** (released this week) restructured elicitation into exactly this
  shape at the protocol level: a server returns `InputRequiredResult` with an opaque
  `requestState`; the client gathers the human's answer and re-issues the call — a
  stateless pause/resume. **URL-mode elicitation** covers interactions that must happen
  outside the agent client (OAuth consent, credential entry) —
  https://blog.modelcontextprotocol.io/posts/2026-07-28/ ;
  https://stacktr.ee/blog/mcp-2026-spec-changes ;
  https://nerdleveltech.com/mcp-stateless-protocol-enterprise-authorization
  The spec deliberately leaves token vaulting, JIT consent, RBAC, and audit to the
  runtime layer — i.e., the bus owns them.

**[inference — F1]** This is the release's core enforcement design and its cleanest
differentiator: competitors bolt approvals onto agent frameworks; Greenwood can make the
approval a *structural precondition of the event fold*. Spec it now — the envelope and
stream shapes are the expensive-to-change part.

## 4. Authority propagation: the originating human in the envelope

Source (July 6, 2026): https://aws.amazon.com/blogs/security/enforce-least-privilege-authorization-in-multi-agent-ai-chains-using-cedar/
[verified] — addresses **OWASP ASI03** (Identity & Privilege Abuse from the new agentic
top-10): across delegation hops "an agent can potentially act beyond what the originating
user authorized."

Mechanics worth stealing:

- The originating user's verified claims (`sub`, `role`, `amr`/MFA, `session_id`) travel
  with the request as **integrity-protected context** (HMAC-SHA256 over the user context;
  every downstream evaluator verifies before trusting).
- Per-hop **OAuth Token Exchange (RFC 8693)** mints scoped on-behalf-of tokens recording
  "who's acting on behalf of whom and with what authority."
- Three Cedar layers, halt on first deny: L1 agent→tool (trust level, namespace,
  lifecycle stage), L2 agent→agent (hop count ≤ N, requested tasks ⊆ target
  capabilities), L3 originating-user (role, MFA, delegation depth). Key design note:
  "the principal remains the agent, not a user entity" — the human is checked via
  context attributes. Their test case shows why L3 exists: a support-role user's deletion
  request passes L1 and L2 and is denied only by L3.

**[inference — F6, F13]** For the release, the full three-evaluator apparatus is
oversized — but the **envelope field is not**: D8 already makes messaging the inter-agent
primitive with per-message governance, so add `originating_principal` (human id, role,
auth strength, session) as a signed envelope field from day one, and write the one rule
that matters for the junior-operator story: *no delegated action may exceed the tier the
originating human could authorize directly*. Hop-count limits and per-hop token exchange
are M2+.

## 5. Credential brokering: the Vault-validated pattern [verified]

Source: https://developer.hashicorp.com/validated-patterns/vault/ai-agent-identity-with-hashicorp-vault
(HashiCorp validated pattern; product framing at
https://www.hashicorp.com/en/products/vault/use-cases/agentic-runtime-security and
https://www.hashicorp.com/en/blog/secure-ai-identity-with-hashicorp-vault):

- User authenticates (Entra ID) → agent performs **OAuth OBO token exchange** so the
  token carries *both* agent identity and user claims (end-to-end traceability — same
  principle as §4, applied to secrets).
- The MCP server (not the agent) validates the OBO token and authenticates to Vault via
  **JWT auth**; group claims map to Vault external groups → policies. The LLM never
  touches Vault; "the MCP server encapsulates Vault and API/service logic." No separate
  broker component.
- Vault issues short-lived auth tokens and **dynamic time-bound credentials** (e.g.,
  DB passwords minted per use, auto-revoked; `token_ttl`/`token_max_ttl` configurable):
  https://oneuptime.com/blog/post/2026-02-20-vault-dynamic-secrets/view
- **`X-Correlation-ID` threading** across Web → Agent → MCP Server → Vault so one id
  joins logs across the whole stack.

**[inference — F7]** Confirms pass-1 §4 with a named, vendor-validated reference
architecture. For Aug 16: operating Vault is not release-sized; keep the trellis-style
credential proxy with per-task TTL scoping. The two cheap steals: (a) the *tool
executor*, not the agent, is the only credential-holding component; (b) correlation is
end-to-end — which the bus provides natively via event causality, provided credential
issuance itself is logged as a bus event. Vault + OBO is the M2 upgrade path; secret
*storage* on bc-prod stays sealed-secrets (per platform skills) until then.

## 6. Platform developments pass 1 missed

### Claude Code PreToolUse hooks: veto that survives bypass [reported, multiple consistent sources]

Hooks (early 2026) run user-defined commands at lifecycle points; PreToolUse can return
allow/ask/deny with "absolute veto power" — and critically, **hooks and deny rules still
enforce under `bypassPermissions` / `--dangerously-skip-permissions`** (bypass skips only
interactive confirmations); "a deny at any settings level cannot be allowed by another
level": https://hidekazu-konishi.com/entry/claude_code_hooks_complete_guide.html ;
https://www.morphllm.com/claude-code-hooks ;
https://pasqualepillitteri.it/en/news/1832/claude-code-dangerously-skip-permissions-pretooluse-hooks-2026 ;
https://blakecrosley.com/blog/claude-code-hooks-explained

**[inference — F4]** Pass 1 flagged trellis's `bypassPermissions` default as the release's
worst hole. This is the shipped fix at the runtime layer: a PreToolUse hook that consults
the bus's Cedar policy (or requires a bus-issued execution grant) enforces the tier matrix
*even in bypass mode*. Defense in depth: fold-level gate (F1) is authoritative;
runtime-hook gate stops the tool call before it leaves the agent host.

### Microsoft Agent Governance Toolkit [verified]

https://developer.microsoft.com/blog/securing-mcp-a-control-plane-for-agent-tool-execution/
— a control plane between MCP clients and tool servers. Three checkpoints: tool-definition
scanning (hidden instructions/typosquatting) before definitions reach the model; per-call
deterministic policy (YAML/Rego/Cedar) outputting **allow / deny / require-approval**;
response-content validation before results return to the agent. Plus: SPIFFE-compatible
agent identities (Ed25519 + ML-DSA-65), append-only hash-chained audit with replay
debugging, kill switches, and a **0-1000 trust score that decays on violations and
recovers with compliant behavior**.

**[inference — F9, F14]** The allow/deny/require-approval triple matches the garden plan.
The trust score is the mechanized version of pass-1's ITIL standard-change promotion —
release keeps promotion as a human (Terra) decision; a computed per-action-type track
record is the M2+ input to that decision, not a replacement for it. Tool-definition
scanning is LOW while the garden authors all its own tools.

### OpenAI AgentKit / Connector Registry [verified/reported]

https://openai.com/index/introducing-agentkit/ ;
https://developers.openai.com/api/docs/guides/agents/guardrails-approvals ;
https://osher.com.au/blog/openai-connector-registry/ — Agents SDK cleanly splits
**guardrails** (automatic checks) from **human approvals** (paused runs); the Connector
Registry adds an org-level governance object: which connectors/MCP servers exist, who
approved their admission, who may call them.

**[inference — F12]** Validates a garden **tool registry with approval provenance** —
each tool's entry records its tier classification, who admitted it, and when. Release:
a hand-curated list in config with those fields. M2: a real registry.

### incident.io Intelligent Runbook Execution [reported]

https://incident.io/blog/runbook-automation-tools-2026-the-complete-guide — approval
gates as steps *inside* runbook flows; approver routing driven by a live catalog
(alert→service→team→on-call) instead of hardcoded names; audit trails auto-generated;
practitioner consensus: human gates on "any step that touches production systems or
mints secrets." PagerDuty ships the same HITL-steps-in-automation shape:
https://www.pagerduty.com/platform/automation/workflow/

**[inference — F11]** With one operator the catalog routing is moot; the durable shape
for M2 is "remediation = runbook with typed steps, some of which are gates" — which maps
naturally onto a bus saga whose gate steps are F1 approval events.

## 7. Open questions raised by this pass

1. **Agent-as-maker semantics**: maker-checker literature assumes human makers. When the
   maker is an agent, does the *proposing agent's* identity matter for segregation (e.g.,
   may a different agent of the same garden approve? — answer should be no; only
   principals with human authority approve), and how is that expressed in Cedar? Record
   as a spec decision, not researched practice.
2. **Cedar entity model for the garden**: what are principals (agents? agent+originating
   human?), actions (tool calls? bus commands?), resources (apps? environments?
   topics?). The AWS pattern ("principal is the agent; human in context") is one answer;
   needs a design session.
3. **Approval expiry defaults**: fintech uses 24/48/72h reminder/escalate/expire; right
   TTLs for ops actions are probably much shorter (minutes-hours). Unresearched — pick
   defaults and label them.
4. **Where exactly the Cedar check runs** relative to the decider (inside `decide` vs a
   bus-side interceptor emitting the proposal): both preserve the fold invariant; inside
   `decide` is purer, interceptor is more reusable across streams. Design call.
