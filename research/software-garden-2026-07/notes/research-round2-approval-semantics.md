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
must decide and label it.

Research note: a mid-session web-tool outage limited some fetches; anything that could
not be primary-verified is labeled [reported] and flagged inline.

---

## Scored findings

| # | Finding | Score | Why |
|---|---------|-------|-----|
| G1 | **Entra PIM forbids non-human approvers outright**: "Approvers aren't able to approve their own role activation requests. Additionally, **service principals aren't allowed to approve requests**." | **HIGH** | The clearest shipped-system statement of the Q1 position: approval authority is human-only; automated identities are structurally barred from the checker role. Cite it in the spec as precedent for "no agent approves an agent." |
| G2 | **GitHub Actions required reviewers are humans/teams only**; bots can't be added as reviewers; the only "automation approves" path is a human's PAT (i.e., a human credential) or an explicitly-registered GitHub-App *deployment protection rule* — a deterministic gate, not a reviewer | **HIGH** | Second shipped-platform confirmation of the same split: *reviewers are humans; automation may only participate as an explicitly configured deterministic protection rule*. Gives Greenwood the exact vocabulary: agents can be gate *inputs* (checks), never gate *approvers*. |
| G3 | **SoD literature: a single AI agent that initiates+authorizes+executes collapses segregation of duties**; SOX-oriented guidance calls AI auto-clearance without independent human review an SoD violation | **HIGH** | Grounds the T3/T4 rule in control language auditors recognize: when the maker is an agent, the checker must be an independent principal with human authority — and "another agent of the same system" fails the independence test because both share operator/model/prompt lineage. |
| G4 | **AgentCore's Cedar entity model: the principal is the authenticated caller identity** (`AgentCore::OAuthUser` from the inbound JWT `sub`, or `AgentCore::IamEntity`), actions are auto-generated per tool (`Target___operationId`), resource = gateway, tool params in `context.input.*` | **HIGH** | The published production Cedar-for-agents model. Combined with the AWS multi-agent blog (pass 1: principal = agent, originating human as integrity-protected context), it fixes the two viable entity models for Greenwood and what each buys. |
| G5 | **Ops approval/elevation TTLs cluster tightly**: PIM pending-approval window = 24 h fixed (non-configurable), elevation default 8 h (1–24 h); Teleport pending-request TTL default 1 h → 1 week (v15), approved elevation ≤ 14 days (`max_duration`), and Teleport explicitly separates *request-pending TTL* from *elevated-session TTL*; GitHub Actions runs wait ≤ 30 days for review [reported] | **HIGH** | Q3 answered with shipped numbers: pending-approval windows are hours-to-days; elevation/execution windows are minutes-to-hours-to-a-day. Greenwood's third TTL (granted-but-unexecuted approval) has *no* shipped precedent — systems consume approvals immediately — so short defaults are a design call, but the two adjacent TTLs are well-precedented. |
| G6 | **CQRS practice puts authz in a dispatch interceptor before the handler, and anything state-dependent inside the aggregate**: recommended flow is Command Validator → Authorization Middleware → Command Handler; Axon ships `CommandDispatchInterceptor` (role checks on annotated commands) + in-aggregate checks for state-dependent rules; "never authorize from read-model data" | **HIGH** | Direct prior art for Q4: the community's split is *identity/role checks bus-side (interceptor), invariant checks fold-side (aggregate)*. Maps 1:1 onto Greenwood: Cedar tier check in a bus-side interceptor that emits the proposal/auto-approval event; the decider enforces the approval-precondition invariant. |
| G7 | Replay-determinism prior art for "policy decision as recorded event" is **thin to absent**: no named pattern found for versioned policy-as-events; the grounding is the Decider/ES first principle that nondeterministic inputs are resolved at command-handling time and only their *results* are folded | MEDIUM | Confirms §7-Q4's replay half is a **design call** — but one derivable from ES first principles rather than invented: record `policy_version` + decision in the event; `evolve` never calls Cedar. |
| G8 | Axon community wrinkle: security context is lost when sagas dispatch follow-up commands asynchronously — authentication must be explicitly carried on the message | MEDIUM | Exactly the failure Greenwood's D8 envelope `originating_principal` field prevents; use as a test case (saga-issued command must still carry and re-verify origin). |
| G9 | GitHub Apps deployment protection rules: max 6 enabled per environment; any number installable | LOW | Sizing detail only; confirms "automated gates are registered objects," nothing else Greenwood needs. |
| G10 | PIM approval mechanics: first responder resolves the request; approvers need no role themselves; ≥2 approvers recommended; admins can cancel any pending request | MEDIUM | Small shipped semantics worth copying into the approval aggregate: first-decision-wins, separate cancel authority, approver set need not hold the privilege being granted. |

---

## 1. Q1 — Agent-as-maker: who may approve? (segregation semantics)

### Shipped platforms bar non-human approvers

**Entra PIM** [verified —
https://learn.microsoft.com/en-us/entra/id-governance/privileged-identity-management/pim-approval-workflow]:

> "Approvers aren't able to approve their own role activation requests. Additionally,
> **service principals aren't allowed to approve requests**."

Two segregation rules in one sentence: no self-approval, and — decisive for Q1 — *no
non-human identity may hold the checker role at all*. This is the strongest shipped
statement found of "only principals with human authority approve." Also useful: the
first approver to respond resolves the request; approvers do not need to hold any role
themselves; Microsoft recommends ≥2 configured approvers (availability, not quorum).

**GitHub Actions environments** [reported — search summaries over
https://docs.github.com/en/actions/reference/workflows-and-actions/deployments-and-environments,
https://github.com/orgs/community/discussions/63129,
https://github.com/orgs/community/discussions/65651; primary-page fetch blocked by the
web outage]:

- Required reviewers for a protected environment are **users or teams** — bots (e.g.,
  Dependabot) cannot be added as required reviewers (long-standing community ask,
  unfulfilled).
- The workflow's own `GITHUB_TOKEN` cannot approve a deployment; the documented
  "auto-approval" workaround is a **PAT belonging to a human who is a required
  reviewer** — i.e., automation can only approve by *wielding a human's credential*,
  which is delegation of a human's authority, not bot authority. (Marketplace actions
  like `automate-environment-deployment-approval` are exactly this.)
- The sanctioned path for automated gating is different in kind: **GitHub-App-backed
  custom deployment protection rules** — a registered, deterministic check object (max 6
  enabled per environment), not a "reviewer." GitHub also ships a *prevent self-review*
  environment option for the human path.

**[inference]** The two platforms agree on a taxonomy Greenwood should adopt verbatim:

| Role | Who may hold it |
|---|---|
| Maker / proposer | human or agent |
| **Checker / approver** | **human principal only** (or a human's explicit credential) |
| Automated gate | allowed, but as a *registered deterministic protection rule* (Cedar policy, CI check) — configured in advance by a human, evaluated mechanically, never an LLM judging per-case |

An LLM agent is neither: it is not a human principal, and it is not deterministic, so it
can occupy no approval-side role. "A different agent of the same garden approves" is not
supported by any shipped platform found.

### SoD / SOX literature: independence fails between co-tenant agents

- CloudEagle, "Why AI Agents Break Segregation of Duties Controls"
  (https://www.cloudeagle.ai/blogs/segregation-of-duties-ai-agents) [reported]: a single
  AI agent can hold initiation, authorization, and execution at once; SoD "has nothing
  to enforce a boundary against when the actor is a single automated process."
- SOX 404 AI-assisted-controls guidance (https://www.finrep.ai/blog/sox-404-compliance-checklist-for-ai-assisted-controls-2026)
  [reported]: "auto-clearance without human approval is an SoD risk if the AI system can
  both detect and resolve an exception without independent review."
- SoD platforms now explicitly extend identity governance to "service accounts, APIs,
  and AI agents" as *governed* identities (https://www.safepaas.com/segregation-of-duties/segregation-of-duties-in-modern-enterprise-systems-identity-risk-and-control/)
  [reported] — governed as actors, not empowered as approvers.

**[inference]** The SoD independence test explains *why* agent-approves-agent fails even
with distinct agent identities: two agents in the same garden share an operator, a model
vendor, prompt lineage, and (often) a compromise surface — a prompt-injection that
reaches the maker plausibly reaches the checker. They are one "party" in two-person-rule
terms (NIST AC-3(2), pass 2 §2). Distinct agent IDs give attribution, not independence.

**Position for the spec** — industry-aligned, write as a Cedar-enforceable invariant:

1. Approval (`ApprovalGranted`) is valid only from a principal of type `Human`
   (session-authenticated, envelope-verified). Agents cannot emit it; the decider
   rejects it from any non-human principal — Greenwood's equivalent of PIM's
   "service principals aren't allowed to approve."
2. `checker != maker` **and** `checker != originating_principal of the proposal` for T4
   (the junior can't two-person-approve what the junior initiated — matches GitHub's
   prevent-self-review and PIM's no-self-approval).
3. Agents *may* contribute machine checks (tests green, canary healthy, Cedar permit)
   as **evidence attached to the proposal** — the protection-rule role — and a tier
   policy may *require* such evidence, but evidence never substitutes for the human
   `ApprovalGranted` event at T3/T4.

## 2. Q2 — Cedar entity modeling for the garden

### The two published models

**Model A — principal = authenticated caller (human), agent is the channel.**
Bedrock AgentCore Policy [verified —
https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/policy-core-concepts.html]:

- Principals: `AgentCore::OAuthUser` (built from the inbound JWT `sub`; JWT claims —
  username, role, scope — become principal **tags**) or `AgentCore::IamEntity` (caller's
  IAM ARN). There is **no first-class Agent principal type**: when the gateway is called
  with the human's OAuth token, the *human* is the principal and the agent is invisible
  to policy; when called with the agent runtime's IAM role, the *agent's role* is the
  principal and the human is invisible.
- Actions: auto-generated from tool definitions, one per operation
  (`<TargetName>___<operationId>`); the schema is derived from the tool specs, so
  policies validate against real tool signatures at authoring time.
- Resource: the gateway (`AgentCore::Gateway::"<arn>"`) — coarse.
- Tool parameters land in `context.input.*`; gateway callers can inject extra
  `context.<custom>` attributes.
- Common patterns [verified —
  https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/policy-common-patterns.html]:
  emergency-shutdown `forbid(principal, action, resource)`; per-tool forbid; RBAC via
  `principal.getTag("role")`; parameter conditions (`context.input.amount < 500`,
  `context.input has shippingAddress`). Example-policies page shows the HITL boundary
  pattern: permit the autonomous path below a threshold, `forbid` the agent principal
  from the high-value action entirely so it must hand off [reported —
  https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/example-policies.html,
  https://medium.com/@tolghn/amazon-bedrock-agentcore-policy-cedar-control-what-your-ai-agents-can-do-3313e6f3a2db].

**Model B — principal = agent, originating human as integrity-protected context.**
The AWS multi-agent-chains blog (pass 2 §4, verified there): principal stays the agent
entity; the originating user's claims travel as HMAC-protected context checked by a
dedicated policy layer ("the principal remains the agent, not a user entity").

**[inference]** The choice tracks *whose authority is being spent*. Model A cannot
express agent-specific ceilings (you can't forbid "agent X" when agent X isn't an
entity); Model B keeps the agent governable but must re-verify human context integrity
at every evaluation. No third model appears in the wild; open-source examples of
tool-calls-as-Cedar-actions beyond AgentCore's auto-generated schema were not found
before the web outage — the AgentCore schema convention (one action per tool operation,
params in `context.input`) is the de-facto reference [reported].

### Proposed Greenwood entity model [design call, grounded in A+B]

Greenwood needs both parties to be first-class (agents have tier ceilings *and* humans
have tier authority), which is Model B plus typed principals:

```cedar
// entity types
Garden::Human   { role: String, max_tier: Long, auth_strength: String }
Garden::Agent   { tier_ceiling: Long, lifecycle: String }   // agent's own trust ceiling
// actions: one per tool operation, AgentCore-style "Target___operationId",
//   each with annotation/attribute required_tier: Long
// resources: Garden::App > Garden::Environment (hierarchy; membership via `in`)
// context: input.* (normalized tool params), originating_principal
//   { id, role, max_tier, auth_strength, session_id } — envelope-verified (D8/DC-3)
```

The §7-Q1 rule and the tier-ceiling invariant become one readable policy pair:

```cedar
// no delegated action exceeds the originating human's tier (OWASP ASI03)
forbid (principal is Garden::Agent, action, resource)
when { context.action_tier > context.originating_principal.max_tier };

// nor the agent's own earned ceiling
forbid (principal is Garden::Agent, action, resource)
when { context.action_tier > principal.tier_ceiling };
```

so the effective tier is min(agent ceiling, originating human authority) by two forbids
— no arithmetic needed, and each denial names which ceiling was hit. Approval-side:
`ApprovalGranted` is itself a bus command whose Cedar check requires
`principal is Garden::Human && principal.max_tier >= context.action_tier` (Q1 rule 1).
Steal from AgentCore regardless of model: schema **generated from tool definitions**
(policies validate against real signatures), and the standing
`forbid(principal, action, resource)` emergency-stop policy slot.

## 3. Q3 — Approval expiry TTLs in shipped ops systems

Three distinct clocks; shipped systems name only two of them:

**Clock 1 — pending-approval window** (request sits unreviewed):

| System | Default | Range | Source |
|---|---|---|---|
| Entra PIM role activation | **24 h, fixed** ("isn't configurable"; expired ⇒ re-submit) | — | [verified] https://learn.microsoft.com/en-us/entra/id-governance/privileged-identity-management/pim-approval-workflow |
| Teleport access request | 1 h if unset → **default raised to 1 week** in v15 (PR #39509) | configurable | [reported] https://github.com/gravitational/teleport/pull/39509 ; https://goteleport.com/docs/identity-governance/access-requests/access-request-configuration/ |
| GitHub Actions environment review | run fails if unreviewed ≈ **30 days** | — | [reported; primary fetch blocked] |
| Fintech maker-checker (pass 2) | 24 h remind / 48 h escalate / **72 h expire** | — | pass 2 §2 |

**Clock 2 — elevation/session window** (approved privilege lives):

| System | Default | Max | Source |
|---|---|---|---|
| Entra PIM activation | **8 h** | 1–24 h | [verified] https://learn.microsoft.com/en-us/entra/id-governance/privileged-identity-management/pim-how-to-change-default-settings |
| Teleport approved request | bounded by session TTL | `max_duration` ≤ **14 days** | [reported] https://goteleport.com/docs/identity-governance/access-requests/access-request-configuration/ |

Teleport is the system that most explicitly **separates the two clocks** — issues
#14040/#16202 are users asking for exactly the request-TTL vs session-TTL split
(https://github.com/gravitational/teleport/issues/16202) [reported].

**Clock 3 — granted-but-unexecuted approval validity.** No shipped precedent found:
PIM, Teleport, and CI/CD gates all *consume the approval immediately* (activation
starts, the run resumes) — the "approval sits granted, execution comes later" gap only
exists in maker-checker systems, and fintech's 72 h horizon is calibrated to business
transactions, not ops. PagerDuty/incident.io approval-step timeout defaults could not
be primary-verified before the web outage; nothing found contradicts the picture above.
**[design call]** — this is Greenwood's own clock to pick.

**Proposed defaults [design call, labeled per §7-3]** — versioned policy numbers
(DC-5 style), per tier:

- **Pending window**: T3 4 h (junior is the approver and online), T4 24 h (two humans;
  matches PIM's fixed window); expiry emits `ProposalExpired`, never executes.
- **Approval validity (clock 3)**: T3 **15 min**, T4 **60 min** from `ApprovalGranted`
  to `ExecuteAction`, on the rationale that the approval was granted against a live
  system state (target_version) whose shelf life is minutes — and edit-invalidation
  already voids it on any target change. Longer-running preparation belongs *before*
  proposal, not between approval and execution.
- **Execution/elevation window**: cap any credential or grant minted for the approved
  action at the action's expected duration, ≤ 8 h (PIM's default as the outer bound).

## 4. Q4 — Where the Cedar check runs (decider vs interceptor) and replay

### Community practice: split by check type

- General CQRS guidance [reported —
  https://reintech.io/blog/implementing-authorization-security-cqrs-environment]:
  recommended pipeline is `Command Validator → Authorization Middleware → Command
  Handler` — identity/permission checks live in middleware *before* the handler; and
  **never authorize against read-model data** (the projection lag is an exploitable
  TOCTOU window). Aligned: https://www.cqrs.com/event-driven-architecture/command-handlers/.
- Axon practice [reported —
  https://discuss.axoniq.io/t/best-practices-for-applying-spring-security-for-domain-object-authorization/3501
  and axonframework Google-group threads]: role/identity checks via a
  `CommandDispatchInterceptor` over annotated command classes; **"if you need to
  validate anything that requires aggregate state, you must do so within the aggregate
  itself."** Known wrinkle: sagas dispatching follow-up commands lose the security
  context — it must be carried on the message explicitly (exactly what the D8
  `originating_principal` envelope field fixes).
- EventStoreDB/Emmett-specific write-model-authz writing: none found before the outage;
  no evidence the answer differs there. [reported/absence]

**[inference]** Translated to Greenwood's terms, the community split is:

- **Bus-side interceptor (dispatch-time): the Cedar tier check.** It is identity- and
  parameter-based, needs no fold state, and is reusable across every stream. It runs
  once per proposal and its *outcome* is emitted as an event
  (`ActionAutoApproved{policy_version, decision_id}` or `ApprovalRequested{...}`).
- **Decider (fold-side): the approval invariant.** "No `ExecuteAction` without a
  matching, unexpired, parameter-bound `ApprovalGranted` in folded state" is
  state-dependent, so by Axon's own rule it belongs in the aggregate/decider — it is a
  domain invariant, not an authz middleware concern.

This resolves §7-Q4's "purer vs more reusable" tension: it isn't either/or — practice
puts the *policy evaluation* bus-side and the *invariant* fold-side, which is also the
defense-in-depth reading of pass 2 F1+F4.

### Replay determinism: policy decisions as events [design call, grounded]

No published pattern named "policy decision as recorded event / versioned
policy-as-events" was found (searches interrupted by the outage, but nothing surfaced
in the completed ones either) — score this **design call, no external practice found**,
derived from ES first principles rather than copied:

- The Decider contract (pass 2 §3) makes `evolve` a pure fold; anything nondeterministic
  or environment-dependent (clock, external policy engine, model output) must be
  resolved at *command-handling time* and only its **result** recorded.
- Therefore: `evolve` **never calls Cedar**. The interceptor evaluates Cedar once,
  stamps `policy_version` (content hash or sequence number of the policy set) and the
  decision into the emitted event, and replays fold those events verbatim — a replay
  under a *newer* policy set reproduces the historical decision because the decision is
  data, not re-computation.
- Versioned policy-as-events, cheap release version: policy changes are themselves bus
  events (`PolicySetUpdated{version, content_hash, author}`) on a config stream — which
  gives "policy version stamped into every audit record" (pass 1) plus a replayable
  history of *what the rules were when*, without any extra infrastructure.
- Expiry needs the same treatment: "unexpired" must be judged against event-time
  (approval timestamp + TTL vs the *command's* timestamp, both recorded), never
  wall-clock-at-fold, or replays diverge.

---

## Proposed answers to the four questions

1. **Agent-as-maker segregation** — Industry position confirmed: **only human
   principals approve; no automated identity holds the checker role** (PIM bars service
   principals from approving [verified]; GitHub required reviewers are humans/teams,
   bots excluded [reported]). Agents of the same garden fail the SoD independence test
   (shared operator/model/compromise surface). Agents participate on the approval side
   only as *registered deterministic protection rules* / machine evidence attached to
   the proposal. Spec: `ApprovalGranted` valid only from `Garden::Human`;
   `checker != maker` and, at T4, `checker != originating_principal`.
2. **Cedar entity model** — Adopt Model B with typed principals: `Garden::Human` and
   `Garden::Agent` both first-class; actions auto-generated per tool operation
   (AgentCore convention) each carrying `required_tier`; resources = app/environment
   hierarchy; normalized params in `context.input`; envelope-verified
   `originating_principal` in context. Tier law as two forbids: action tier may exceed
   neither `principal.tier_ceiling` nor `context.originating_principal.max_tier` —
   effective authority is the min of the two, and each denial names its ceiling.
3. **Approval TTLs** — Three clocks. Shipped numbers exist for two: pending-approval
   windows run hours-to-days (PIM 24 h fixed; Teleport 1 h → 1 week default; fintech
   72 h), elevation windows run hours (PIM 8 h default, ≤ 24 h; Teleport ≤ 14 days
   max). The third — granted-but-unexecuted approval validity — has **no shipped
   precedent** (systems consume approvals instantly): design call. Propose T3 pending
   4 h / validity 15 min; T4 pending 24 h / validity 60 min; minted-credential window
   ≤ 8 h; all as versioned policy numbers.
4. **Gate placement** — Community practice (CQRS middleware guidance + Axon): put the
   **Cedar evaluation in a bus-side dispatch interceptor** (identity/parameter check,
   reusable, no fold state) and the **approval invariant in the decider** (state-
   dependent ⇒ in-aggregate, per Axon's rule). Replay: `evolve` never calls Cedar —
   the interceptor's decision is recorded as an event stamped with `policy_version`,
   policy changes are themselves events, and expiry is judged on recorded timestamps,
   so folds are deterministic under any later policy set. The recorded-decision half is
   a design call grounded in Decider purity; no named external pattern found.
