---
title: "Round 2: approval semantics — agent-as-maker segregation, Cedar entity modeling, approval TTLs, gate placement in the fold"
date: 2026-07-29
topic: answers to the four open questions from research-emergent-2-deep.md §7 (DC-1/DC-2 inputs)
status: draft
---

Targeted follow-up pass. Researches **only** the four open questions from
`research-emergent-2-deep.md` §7 — nothing below repeats that file or pass 1. Labels:
**[verified]** = fetched primary page; **[reported]** = search-result summaries;
**[inference]** = synthesis; **[design call]** = literature genuinely silent, Greenwood
must decide and label it. (A mid-session web-tool outage delayed some fetches; every
load-bearing claim below was eventually primary-verified or is labeled [reported].)

---

## Scored findings

| # | Finding | Score | Why |
|---|---------|-------|-----|
| G1 | **Entra PIM forbids non-human approvers outright**: "Approvers aren't able to approve their own role activation requests. Additionally, **service principals aren't allowed to approve requests**." | **HIGH** | The clearest shipped-system statement of the Q1 position: approval authority is human-only; automated identities are structurally barred from the checker role. Cite as precedent for "no agent approves an agent." |
| G2 | **GitHub Actions required reviewers are users/teams only** (≤6; one approval suffices; *prevent self-review* option); bots can't be reviewers; automation approves only via a human's PAT (a human credential) or as an explicitly registered GitHub-App *deployment protection rule* — a deterministic gate, not a reviewer; unreviewed runs fail after a non-configurable 30 days | **HIGH** | Second shipped-platform confirmation of the same split: *reviewers are humans; automation participates only as a pre-registered deterministic protection rule*. Gives Greenwood the vocabulary: agents can be gate *inputs* (checks/evidence), never gate *approvers*. |
| G3 | **Faramesh (arXiv 2601.17744)**: non-bypassable "Action Authorization Boundary" between agents and tools; intents canonicalized (Canonical Action Representation), evaluated to **PERMIT/DEFER/DENY**; **append-only decision provenance keyed by canonical action hashes enabling deterministic replay without re-running agent reasoning** | **HIGH** | Direct academic prior art for DC-1's replay half: authorization decisions recorded as provenance records so replay re-folds decisions instead of re-evaluating policy or re-running the agent. DEFER = the approval-pending state. Validates proposal-hash binding too. |
| G4 | **Zylos "Replayable Agent Runtimes"**: event-sourced agent execution with `tool.proposed` → `tool.authorized` (records "policy decision, human approval, or denial") and `interrupt.raised/resolved` as first-class events; committed side effects replay from recorded results; a "policy bundle" attaches at run creation | **HIGH** | Independent publication of Greenwood's exact gate-in-the-fold event shape (`ActionProposed → ApprovalGranted/Denied`), including approvals-as-events and replay determinism via recorded outcomes. DC-1 is no longer a lone invention — converging practice. |
| G5 | **AgentCore's Cedar entity model (Model A): principal = authenticated caller** (`AgentCore::OAuthUser` from JWT `sub` with claims as tags, or `AgentCore::IamEntity`); actions auto-generated per tool (`Target___operationId`); resource = gateway; tool params in `context.input.*`; no Agent principal type exists | **HIGH** | One of the two published production models, and the source of two steals regardless of model: schema generated from tool definitions, and a standing `forbid(principal, action, resource)` emergency-stop slot. |
| G6 | **AWS multi-agent Cedar sample (Model B, verified at repo level)**: entities `Agent` (trust_level, namespace, lifecycle_stage, capabilities) and `Tool` (namespace, risk_level); actions `invoke_tool`, `delegate_task`; **no User entity — human flows as `context.originating_user`** (JWT-derived id/role/mfa/session); "human authorization never grants permissions — it only constrains what agents can do" | **HIGH** | The published answer to "what entity model expresses 'no delegated action exceeds the originating human's tier'": agent-as-principal, human-as-constraining-context, ceilings as forbids. Directly reusable for the T1–T4 matrix. |
| G7 | **Ops approval/elevation TTLs cluster tightly**: PIM pending window **24 h fixed**, elevation default **8 h** (1–24 h); Teleport pending default 1 h → **1 week** (v15+), elevation ≤ **14 days**, with explicit request-TTL vs session-TTL separation; GCP PAM grants **30 min – 7 days**; GitHub Actions review wait **30 days fixed**; fintech maker-checker 72 h | **HIGH** | Q3 answered with shipped numbers: pending windows are hours-to-days, elevation windows minutes-to-hours-to-a-day. The third clock (granted-but-unexecuted approval validity) has **no shipped precedent** — systems consume approvals immediately — so Greenwood's short defaults are a design call flanked by good numbers. |
| G8 | **CQRS/Axon practice splits authz by check type**: identity/role checks in a dispatch interceptor *before* the handler; "anything that requires aggregate state must be validated within the aggregate"; never authorize from read-model data | **HIGH** | Resolves §7-Q4's decider-vs-interceptor tension: it isn't either/or — policy evaluation runs bus-side, the approval *invariant* lives fold-side. Maps 1:1 onto Greenwood. |
| G9 | **cedar-policy/cedar-for-agents** (official Cedar org): auto-generates Cedar schemas + authorization requests from **MCP tool descriptions** (Rust/WASM/Python), plus an MCP server exposing Cedar analysis | MEDIUM | Tooling that makes the "actions = tool operations" convention nearly free for Greenwood's MCP-shaped tools; adopt the generated-schema convention now, the crate itself when tools standardize on MCP. |
| G10 | SoD/SOX literature: one AI agent holding initiate+authorize+execute collapses segregation; AI auto-clearance without independent human review is an SoD violation; governance platforms treat AI agents as *governed* identities, never approvers | MEDIUM | Control-language grounding (auditor-recognizable) for why agent-approves-agent fails independence: shared operator/model/prompt lineage = one "party" in two-person-rule terms. |
| G11 | PIM approval micro-semantics: first responder resolves the request; approvers need no role themselves; ≥2 approvers recommended; admins can cancel any pending request | MEDIUM | Copy into the approval aggregate: first-decision-wins, distinct cancel authority, approver set need not hold the privilege being granted. |
| G12 | Axon wrinkle: security context is lost when sagas dispatch follow-up commands — identity must be carried on the message explicitly | MEDIUM | Exactly the failure the D8 `originating_principal` envelope field prevents; becomes a test case (saga-issued command still carries and re-verifies origin). |
| G13 | PagerDuty/incident.io approval-step timeout defaults: not published in accessible docs (deadlines/escalations exist as features, no default numbers found) | LOW | Gap noted; the JIT-access systems (G7) are the better-documented comparables anyway. |

---

## 1. Q1 — Agent-as-maker: who may approve?

### Shipped platforms bar non-human approvers

**Entra PIM** [verified —
https://learn.microsoft.com/en-us/entra/id-governance/privileged-identity-management/pim-approval-workflow]:

> "Approvers aren't able to approve their own role activation requests. Additionally,
> **service principals aren't allowed to approve requests**."

Two segregation rules in one sentence: no self-approval, and — decisive for Q1 — *no
non-human identity may hold the checker role at all*. The strongest shipped statement
found of "only principals with human authority approve."

**GitHub Actions environments** [verified —
https://docs.github.com/en/actions/reference/workflows-and-actions/deployments-and-environments,
https://docs.github.com/en/actions/how-tos/deploy/configure-and-manage-deployments/review-deployments]:

- Required reviewers are **users or teams** (up to six; any one may approve). Bots
  cannot be added as reviewers — a long-standing, unfulfilled community ask
  (https://github.com/orgs/community/discussions/63129) [reported].
- A **prevent self-review** option blocks the run initiator from approving their own
  deployment [verified].
- The workflow's own `GITHUB_TOKEN` cannot approve; the documented "auto-approval"
  workaround uses a **PAT belonging to a human required reviewer** — automation approves
  only by wielding a *human's* credential, i.e. delegated human authority, not bot
  authority (https://github.com/marketplace/actions/automate-environment-deployment-approval)
  [reported].
- The sanctioned automated-gating path is different in kind: **GitHub-App-backed custom
  deployment protection rules** — registered deterministic check objects (max 6 enabled
  per environment), not "reviewers" [reported].
- Unreviewed runs **fail automatically after 30 days**; the limit is not configurable
  (https://github.com/orgs/community/discussions/5673) [reported].

**Faramesh** [verified abstract — https://arxiv.org/abs/2601.17744] encodes the same
stance architecturally: the policy boundary emits PERMIT/**DEFER**/DENY — DEFER hands
the case out of the automated plane entirely; no second agent adjudicates.

### SoD / SOX literature: independence fails between co-tenant agents

- "Why AI Agents Break Segregation of Duties Controls"
  (https://www.cloudeagle.ai/blogs/segregation-of-duties-ai-agents) [reported]: a single
  AI agent can hold initiation, authorization, and execution at once; SoD "has nothing
  to enforce a boundary against when the actor is a single automated process."
- SOX 404 AI-assisted-controls guidance
  (https://www.finrep.ai/blog/sox-404-compliance-checklist-for-ai-assisted-controls-2026)
  [reported]: auto-clearance is an SoD risk "if the AI system can both detect and
  resolve an exception without independent review."
- SoD platforms extend governance *over* AI agents as identities — never *to* them as
  approvers (https://www.safepaas.com/segregation-of-duties/segregation-of-duties-in-modern-enterprise-systems-identity-risk-and-control/)
  [reported].

**[inference]** The independence test explains *why* agent-approves-agent fails even
with distinct agent identities: two agents in one garden share an operator, a model
vendor, prompt lineage, and a compromise surface — a prompt-injection reaching the
maker plausibly reaches the checker. They are one "party" in two-person-rule terms
(NIST AC-3(2), pass 2 §2). Distinct agent IDs buy attribution, not independence.

**Position for the spec** — industry-aligned, Cedar-enforceable:

| Role | Who may hold it |
|---|---|
| Maker / proposer | human or agent |
| **Checker / approver** | **human principal only** (session-authenticated) |
| Automated gate | allowed, but as a *pre-registered deterministic protection rule* (Cedar policy, CI check) — configured by a human in advance, evaluated mechanically; never an LLM judging per-case |

1. `ApprovalGranted` is valid only from a principal of type `Human`; the decider rejects
   it from any non-human principal — Greenwood's "service principals aren't allowed to
   approve."
2. `checker != maker`, and at T4 additionally `checker != originating_principal` of the
   proposal (the junior can't two-person-approve what the junior initiated — GitHub's
   prevent-self-review + PIM's no-self-approval, composed).
3. Agents *may* contribute machine checks (tests green, canary healthy, Cedar permit) as
   **evidence attached to the proposal** — the protection-rule role — and a tier policy
   may *require* such evidence, but evidence never substitutes for the human
   `ApprovalGranted` at T3/T4.

## 2. Q2 — Cedar entity modeling for the garden

### The two published models

**Model A — principal = authenticated caller; the agent is invisible.**
Bedrock AgentCore Policy [verified —
https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/policy-core-concepts.html]:
principals are `AgentCore::OAuthUser` (from the inbound JWT `sub`; claims become
principal **tags**) or `AgentCore::IamEntity` (caller ARN). There is **no Agent
principal type**: with the human's OAuth token the human is the principal and the agent
is unrepresented; with the agent runtime's IAM role, vice versa. Actions are
auto-generated from tool definitions (`<TargetName>___<operationId>`), the resource is
the gateway (coarse), tool parameters land in `context.input.*`. Common patterns
[verified — https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/policy-common-patterns.html]:
standing emergency-stop `forbid(principal, action, resource)`; per-tool forbids; RBAC
via `principal.getTag("role")`; parameter conditions (`context.input.amount < 500`,
`context.input has shippingAddress`). The HITL boundary pattern: permit the autonomous
path below a threshold and `forbid` the agent's path above it, forcing hand-off
[reported — https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/example-policies.html].

**Model B — principal = agent; the human is constraining context.**
AWS multi-agent sample [verified at repo level —
https://github.com/aws-samples/sample-cedar-agentic-ai-authorization; blog in pass 2 §4]:
entities `Agent` (`trust_level`, `namespace`, `lifecycle_stage`,
`registered_capabilities`) and `Tool` (`namespace`, `risk_level`); actions
`invoke_tool` and `delegate_task`; **no User entity** — the originating human flows as
`context.originating_user` (JWT-derived `user_id`, `role`, `mfa_verified`,
`session_id`), integrity-protected (HMAC). Layered forbids: agent trust/namespace
(L1), delegation depth ≤ 3 / hops ≤ 5 (L2), originating-user role+MFA for high-risk
actions (L3). The design sentence worth quoting in the spec: **"human authorization
never grants permissions — it only constrains what agents can do."**

**Tooling** [verified — https://github.com/cedar-policy/cedar-for-agents]: the Cedar
org ships crates/WASM/Python for **auto-generating Cedar schemas and authorization
requests from MCP tool descriptions**, plus an MCP server exposing Cedar analysis. The
"actions = tool operations, schema generated from tool specs" convention is now the
de-facto standard on both AWS's managed and open-source tracks.

**[inference]** The choice tracks *whose authority is being spent*. Model A cannot
express agent-specific ceilings (an unrepresented agent can't be forbidden); Model B
keeps agents governable and is the only published model that can state "delegation
never exceeds the originating human" — as a forbid over context.

### Proposed Greenwood entity model [design call, grounded in B + G5's steals]

Model B, with the human typed rather than anonymous (Greenwood also needs humans as
principals — they send commands and grant approvals on the same bus):

```cedar
Garden::Human { role: String, max_tier: Long, auth_strength: String }
Garden::Agent { tier_ceiling: Long, lifecycle: String }
// actions: one per tool operation (AgentCore/cedar-for-agents convention),
//   each annotated required_tier: Long
// resources: Garden::App > Garden::Environment (hierarchy via `in`)
// context: input.* (normalized params); originating_principal
//   { id, role, max_tier, auth_strength, session_id } — the D8/DC-3 envelope field
```

The §7-Q1 rule and the tier law become two readable forbids — effective authority is
min(agent ceiling, originating human) with each denial naming which ceiling was hit:

```cedar
// no delegated action exceeds the originating human's tier (OWASP ASI03)
forbid (principal is Garden::Agent, action, resource)
when { context.action_tier > context.originating_principal.max_tier };

// nor the agent's own earned ceiling
forbid (principal is Garden::Agent, action, resource)
when { context.action_tier > principal.tier_ceiling };
```

Approval-side, Q1 rule 1 is also one policy: granting requires
`principal is Garden::Human && principal.max_tier >= context.action_tier`. Steal from
Model A regardless: **generate the schema from tool definitions** (policies validate
against real signatures at authoring time) and keep the standing emergency-stop forbid
slot.

## 3. Q3 — Approval expiry TTLs in shipped ops systems

Three distinct clocks; shipped systems name only the first two.

**Clock 1 — pending-approval window** (request sits unreviewed):

| System | Default | Notes | Source |
|---|---|---|---|
| Entra PIM activation | **24 h, fixed** | "isn't configurable"; expired ⇒ re-submit | [verified] https://learn.microsoft.com/en-us/entra/id-governance/privileged-identity-management/pim-approval-workflow |
| Teleport access request | 1 h if unset → **1 week default** (v15) | configurable | [reported] https://github.com/gravitational/teleport/pull/39509 ; https://goteleport.com/docs/identity-governance/access-requests/access-request-configuration/ |
| GitHub Actions env review | **30 days, fixed** | run fails; not configurable | [reported] https://github.com/orgs/community/discussions/5673 |
| Fintech maker-checker | 24 h remind / 48 h escalate / **72 h expire** | | pass 2 §2 |

**Clock 2 — elevation/session window** (approved privilege lives):

| System | Default | Max | Source |
|---|---|---|---|
| Entra PIM activation | **8 h** | 1–24 h | [verified] https://learn.microsoft.com/en-us/entra/id-governance/privileged-identity-management/pim-how-to-change-default-settings |
| GCP PAM grant | requester picks | **30 min – 168 h (7 d)** per entitlement | [reported] https://docs.cloud.google.com/iam/docs/pam-create-entitlements |
| Teleport approved request | bounded by session TTL | `max_duration` ≤ **14 d** | [reported] https://goteleport.com/docs/identity-governance/access-requests/access-request-configuration/ |

Teleport is the system that most explicitly **separates the two clocks** — issues
#14040/#16202 are users demanding exactly the request-TTL vs session-TTL split
(https://github.com/gravitational/teleport/issues/16202) [reported]. PagerDuty and
incident.io expose deadlines/escalations on approval steps but publish no default
numbers [reported — G13].

**Clock 3 — granted-but-unexecuted approval validity.** No shipped precedent found:
PIM, GCP PAM, Teleport, and CI/CD gates all *consume the approval immediately*
(activation/run resumes on approval). The "approval sits granted, execution comes
later" gap exists only in maker-checker systems, whose 72 h horizon is calibrated to
business transactions, not ops. **[design call, no external practice found.]**

**Proposed defaults [design call — versioned policy numbers, DC-5 style]:**

- **Pending window**: T3 **4 h** (the junior is the approver and online), T4 **24 h**
  (two humans; matches PIM's fixed window). Expiry emits `ProposalExpired`; expired
  proposals never execute; re-propose to retry.
- **Approval validity (clock 3)**: T3 **15 min**, T4 **60 min** from `ApprovalGranted`
  to `ExecuteAction`. Rationale: the approval was granted against a live system state
  (target_version) whose shelf life is minutes; edit-invalidation already voids it on
  target change; anything that needs longer preparation belongs *before* proposal.
- **Execution/elevation window**: any credential or grant minted for the approved
  action is capped at the action's expected duration, outer bound **8 h** (PIM's
  default), aligning with the trellis-proxy TTL scoping from pass 2 §5.

## 4. Q4 — Where the Cedar check runs, and replay

### Community practice: split by check type

- General CQRS guidance [reported —
  https://reintech.io/blog/implementing-authorization-security-cqrs-environment]:
  pipeline is `Command Validator → Authorization Middleware → Command Handler` —
  identity/permission checks in middleware *before* the handler; and **never authorize
  against read-model data** (projection lag is an exploitable TOCTOU window).
- Axon practice [reported —
  https://discuss.axoniq.io/t/best-practices-for-applying-spring-security-for-domain-object-authorization/3501]:
  role/identity checks via `CommandDispatchInterceptor` over annotated commands;
  **"if you need to validate anything that requires aggregate state, you must do so
  within the aggregate itself."** Known wrinkle: sagas dispatching follow-up commands
  lose the security context — identity must ride the message (which is precisely the
  D8 `originating_principal` field; make it a test case).
- EventStoreDB/Emmett-specific write-model authz writing: none found; no evidence the
  answer differs there. [reported/absence]

**[inference]** Translated to Greenwood, the split is:

- **Bus-side interceptor (dispatch-time): the Cedar tier check.** Identity- and
  parameter-based, needs no fold state, reusable across every stream. It runs once per
  proposal and its *outcome* is emitted as an event
  (`ActionAutoApproved{policy_version, decision_id}` or `ApprovalRequested{...}`).
- **Decider (fold-side): the approval invariant.** "No `ExecuteAction` without a
  matching, unexpired, parameter-bound `ApprovalGranted` in folded state" is
  state-dependent — by Axon's own rule it belongs in the decider. It is a domain
  invariant, not authz middleware.

So §7-Q4's "purer vs more reusable" tension dissolves: practice puts *policy
evaluation* bus-side and the *invariant* fold-side — which is also the
defense-in-depth reading of pass 2 F1+F4.

### Replay determinism: policy decisions as recorded events

Two 2026 publications now describe exactly this — the pattern has prior art:

- **Faramesh** [verified abstract — https://arxiv.org/abs/2601.17744]: agent intents
  are canonicalized (CAR), evaluated at a non-bypassable boundary to PERMIT/DEFER/DENY,
  and logged in **decision-centric, append-only provenance keyed by canonical action
  hashes**, enabling "auditability, verification, and deterministic replay without
  re-running agent reasoning." Decision provenance records carry policy-version
  identifiers so historical decisions re-fold as data [reported — full text not
  fetched].
- **Zylos "Replayable Agent Runtimes"** [verified —
  https://zylos.ai/research/2026-04-24-replayable-agent-runtimes-event-sourced-execution/]:
  agent runs as append-only event logs with `tool.proposed` (pre-check request),
  **`tool.authorized` ("policy decision, human approval, or denial")**, and
  `interrupt.raised/resolved` for HITL pauses; committed side effects replay from
  recorded results (Temporal-style); a "policy bundle" attaches at run creation.

**[inference — closes §7-Q4]** Greenwood's rules, grounded in the above plus Decider
purity (pass 2 §3):

1. `evolve` **never calls Cedar**. The interceptor evaluates once and stamps
   `policy_version` (content hash of the policy set) + decision id into the emitted
   event; replay folds decisions as data — a replay under a newer policy set still
   reproduces history (Faramesh's property).
2. **Policy changes are themselves bus events** (`PolicySetUpdated{version,
   content_hash, author}`) on a config stream — "policy version stamped into every
   audit record" (pass 1) plus a replayable history of what the rules were when, at
   zero extra infrastructure.
3. **Expiry judged on recorded time**: approval timestamp + TTL compared against the
   *command's* recorded timestamp — never wall-clock-at-fold — or replays diverge.
4. Bind decisions to a **canonical hash of the normalized action** (Faramesh's CAR ≈
   pass 2's params-hash binding): the approval event carries the proposal hash; the
   decider matches on it.

---

## Proposed answers to the four questions

1. **Agent-as-maker segregation** — Industry position confirmed: **only human
   principals approve; no automated identity holds the checker role.** PIM bars service
   principals from approving [verified]; GitHub required reviewers are users/teams only
   with prevent-self-review [verified], and automation approves only via a human's own
   credential; SoD literature says co-tenant agents fail the independence test (shared
   operator/model/compromise surface = one party). Agents participate on the approval
   side solely as *pre-registered deterministic protection rules* / machine evidence on
   the proposal. Spec: `ApprovalGranted` only from `Garden::Human`; `checker != maker`;
   at T4 also `checker != originating_principal`.
2. **Cedar entity model** — Adopt the published Model B (agent = principal, human as
   integrity-protected constraining context — "human authorization never grants, it
   only constrains"), with `Garden::Human` also typed because humans command and
   approve on the same bus. Actions auto-generated per tool operation (AgentCore +
   cedar-for-agents convention) with `required_tier`; resources = app/environment
   hierarchy; tier law as two forbids (action tier ≤ agent ceiling AND ≤ originating
   human's max_tier) so effective authority is their min and each denial names its
   ceiling.
3. **Approval TTLs** — Three clocks; shipped numbers exist for two. Pending-approval:
   hours-to-days (PIM 24 h fixed; Teleport 1 h→1 week; GitHub 30 d; fintech 72 h).
   Elevation: minutes-to-hours (PIM 8 h default/24 h max; GCP PAM 30 min–7 d; Teleport
   ≤14 d). The third — granted-but-unexecuted validity — has **no shipped precedent**
   (approvals are consumed instantly everywhere): design call. Propose T3 pending 4 h /
   validity 15 min; T4 pending 24 h / validity 60 min; minted-credential window ≤ 8 h;
   all as versioned policy numbers.
4. **Gate placement** — Community practice (CQRS middleware + Axon's rule) splits it:
   **Cedar evaluation in a bus-side dispatch interceptor** (stateless, reusable), the
   **approval invariant in the decider** (state-dependent ⇒ in-aggregate). Replay:
   record the decision, never re-evaluate — `evolve` never calls Cedar; events carry
   `policy_version` + canonical action hash; policy changes are events; expiry judged
   on recorded timestamps. Formerly a design call, now grounded: Faramesh's decision-
   provenance replay and Zylos's `tool.proposed/tool.authorized` event schema are 2026
   publications of exactly this shape.
