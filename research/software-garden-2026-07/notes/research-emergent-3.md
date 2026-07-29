# Cost Governance for Agent Fleets: Budgets, Runaway Detection, Spend Attribution

**Research date:** 2026-07-29
**Research question:** What is the state of the art in FinOps for LLM/agent workloads — per-task and per-agent cost attribution, hard budget caps and graceful cutoff behavior, anomaly detection for runaway agent loops, spend forecasting, and the public tooling landscape — and what do documented runaway-agent cost incidents teach about safe defaults?
**Why it matters here:** Trellis treats cost as a first-class operator concern (per-idea dollar figures, Costs dashboard, per-agent budget caps); no other garden component matches it. Greenwood's ~$22k/yr governance-overhead estimate (`research/topics/11-scalability-and-cost.md:171`, `research/decisions.md:89`) is un-grounded in external data. The late-July release's first user is a **junior engineer** operating 4–5 apps who cannot judge whether daily agent spend is normal or a looping agent burning money.

---

## 1. State of AI FinOps in 2026

- AI cost management went from niche to universal fast: two years ago ~31% of FinOps teams managed AI spend; in 2026 it is ~98%, per FinOps X takeaways ([usage.ai](https://www.usage.ai/blogs/finops/ai-ml-cost/finops-x-2026-takeaways)). Gartner forecasts $2.59T global AI spend in 2026.
- The FinOps Foundation now treats AI as a distinct technology category in the Framework, adapting the Inform → Optimize → Operate lifecycle to token billing, GPU scarcity, and inference that scales with adoption ([finops.org framework page](https://www.finops.org/framework/technology-categories/ai/), [FinOps for AI overview WG](https://www.finops.org/wg/finops-for-ai-overview/)). Its core unit-economics metric for API-based spend is **cost per 100K tokens** ([usage.ai KPI playbook](https://www.usage.ai/blogs/finops/ai-ml-cost/practitioner-kpi-playbook/), [finops.org token economics](https://www.finops.org/insights/token-economics-the-atomic-unit-of-ai-value/)).
- The consensus failure mode of *traditional* FinOps on agent workloads (verified across several practitioner sources, e.g. [LeanOps](https://leanopstech.com/blog/finops-for-ai-2026/)): Cost Explorer doesn't break LLM API spend down by feature/customer, reserved-capacity and right-sizing levers don't apply to token pricing, cloud tags don't reach into OpenAI/Anthropic billing, and **cloud anomaly detection misses agentic runaways because they look like normal traffic**. Enforcement has to move to the application/gateway layer.

## 2. Per-task and per-agent cost attribution

### Instrumentation standard: OpenTelemetry GenAI semantic conventions
- The OTel GenAI SIG (active since April 2024) standardizes `gen_ai.*` span attributes: span kinds for LLM calls (`chat`), agent invocations (`invoke_agent`), and tool executions (`execute_tool`), with token counts per span — enabling per-request/per-team/per-feature cost attribution without external billing tooling ([Uptrace](https://uptrace.dev/blog/opentelemetry-ai-systems), [Zylos](https://zylos.ai/research/2026-02-28-opentelemetry-ai-agent-observability/)). Multi-agent conventions (tasks, agent teams, memory) are in active development.
- Key operational insight from this material: **cost and latency correlate with token counts, not request counts.** An agent using 50K tokens for a question that normally takes 3K is the canonical misbehavior signal — invisible without per-span token accounting.

### Application-layer attribution patterns
- **Trace-level (Langfuse):** hierarchical traces capture every LLM call/tool call/retrieval step with token counts and USD cost per generation; filter by user, session, cost, latency, or custom metadata; 6 LLM calls answering one question link into a single trace with per-step cost breakdown ([Langfuse token & cost tracking docs](https://langfuse.com/docs/observability/features/token-and-cost-tracking)).
- **Gateway-level (Helicone, LiteLLM):** proxy intercepts every request and emits cost metrics with no SDK changes; custom properties (user, feature, tenant, agent ID) segment spend for chargeback ([Helicone custom rate limits docs](https://docs.helicone.ai/features/advanced-usage/custom-rate-limits)). LiteLLM tracks spend per virtual key in Postgres; Anthropic's own docs recommend it (unaffiliated) for per-key spend on cloud providers ([Claude Code costs doc](https://code.claude.com/docs/en/costs)).
- **Cloud-native (AWS Bedrock):** *application inference profiles* carry cost-allocation tags (TenantID, ApplicationID, business unit) that flow to Cost Explorer and CUR — but the **finest grain is per usage-type per day, not per request**, tags take up to 24h to appear and are not retroactive ([AWS docs](https://docs.aws.amazon.com/bedrock/latest/userguide/cost-mgmt-application-inference-profiles.html), [AWS ML blog on multi-tenant cost tracking](https://aws.amazon.com/blogs/machine-learning/cost-tracking-multi-tenant-model-inference-on-amazon-bedrock/)). Per-prompt token detail requires per-request metadata tagging in model invocation logs.

**Inference for the garden:** attribution must be captured at the gateway/trace layer at request time (agent ID + task/idea ID as metadata on every call); cloud billing views are a day-late reconciliation layer, never the primary operator view.

## 3. Hard budget caps and graceful cutoff behavior

### Provider-native controls (verified from vendor docs)
- **OpenAI:** two mechanisms — *spend alerts* (notify, traffic continues) and *hard spend limits* at org and project level; hitting a hard limit returns `429` with code `insufficient_quota`. Caveat, verbatim: "Enforcement is not instantaneous, so recorded spend can slightly exceed the configured amount" ([OpenAI spend limits docs](https://developers.openai.com/api/docs/guides/spend-limits)). Note: OpenAI's older "monthly budget" threshold was softened in early 2026 to alert-only ([Alephant](https://blog.alephant.io/openai-spend-limit-how-to-cap-your-api-bill-2026/)).
- **Anthropic:** genuine per-**workspace** monthly spend caps plus workspace rate limits (RPM, input/output TPM); a single Admin API key gives an org-wide usage view across workspaces/models. Claude Code auto-creates a dedicated "Claude Code" workspace for centralized tracking and capping ([Claude Code costs doc](https://code.claude.com/docs/en/costs)).
- **Claude Enterprise (July 2026):** spend limits at three levels — organization, group (RBAC), individual user — with **admin alerts at 75% and 90%** of an org limit and **user-facing notifications at 75% and 95%** of personal limits, plus in-app "request a limit increase" flow ([Anthropic blog](https://claude.com/blog/giving-admins-more-visibility-and-control-over-claude-usage-and-spend), [TechTimes](https://www.techtimes.com/articles/319687/20260704/claude-enterprise-spend-controls-arrive-agentic-ai-bills-blow-past-budgets.htm)).
- **OpenRouter:** org-level guardrails/spending controls exist as well ([OpenRouter guardrails docs](https://openrouter.ai/docs/guides/features/guardrails)).

### Gateway-level enforcement (LiteLLM as reference implementation)
- Per-key/team/user/org/tag/per-window budgets with hard token/request/dollar caps, soft limits with alerts, model whitelists ([LiteLLM budget docs](https://docs.litellm.ai/docs/proxy/users)).
- **Enforcement is genuinely hard to get right** — instructive failure modes from LiteLLM's own tracker: budget checks bypassed when spend already exceeds max_budget ([#26672](https://github.com/BerriAI/litellm/issues/26672)), bypassed when model names don't match `provider/model` format so cost resolves to $0 ([#24770](https://github.com/BerriAI/litellm/issues/24770)). Their answer is a `fail_closed_budget_enforcement` mode: every budgeted request validates against the authoritative DB, so a stale Redis counter cannot under-report spend. Lesson: **budget enforcement must fail closed, and unpriceable requests (unknown model → $0 cost) are a bypass class.**

### Graceful cutoff patterns (practitioner consensus, partly inference)
Layered degradation rather than a single cliff ([TrueFoundry 3-layer gateway](https://www.truefoundry.com/blog/rate-limiting-ai-agents-preventing-llm-api-exhaustion), [Zylos degradation patterns](https://zylos.ai/en/research/2026-02-20-graceful-degradation-ai-agent-systems/)):
1. **Soft threshold** → alert only (human gets time to raise/investigate).
2. **Degrade** → route to cheaper model / cached responses (but track cost-per-successful-request: fallback loops that retry with appended context can drain a monthly budget in hours).
3. **Throttle** → token-bucket rate limiting smooths bursts; circuit breakers stop hammering.
4. **Hard stop** → reject new work with a clear, retryable error (the OpenAI `429 insufficient_quota` pattern); in-flight requests complete so state isn't corrupted.

Everyone's hard-stop semantics are "requests error until limit raised or cycle resets" — nobody documented auto-resuming paused work queues; that part of Trellis's design space is open.

## 4. Runaway detection: anomaly detection for agent loops

### Research: infinite agentic loops are common and structural
"When Agents Do Not Stop: Uncovering Infinite Agentic Loops in LLM Agents" ([arXiv 2607.01641](https://arxiv.org/html/2607.01641v1)) analyzed 6,549 real repos and confirmed **68 infinite-loop failures across 47 projects**. Causes: tool-controlled retries (41.2%), model-dependent termination (38.2% — the agent decides when to stop), missing exits (33.8%), workflow cycles (30.9%), unbounded state growth (27.9%). LangGraph (33.8%) and AutoGen (32.4%) dominate findings. **95.6% of confirmed failures risk API cost exhaustion.** Their IAL-Scan static analyzer (91.9% precision) checks whether feedback paths can repeatedly reach costly operations without an effective bound — the transferable principle: *every feedback path to a paid operation needs a deterministic bound, not a model-judged one.*

### Practical runtime detection signals (practitioner sources)
- **Loop signature in traces:** repeated spans + rising token cost + long runtime + *no state change* ([FutureAGI glossary](https://futureagi.com/glossary/infinite-loop/)).
- **Rate anomaly:** `llm_tokens_used_per_hour` with an IQR algorithm over a ~21-day baseline; worked example flags 450K tokens/hr at 2 AM as a retry loop; "usage today is 300% of normal" alerts ([OpenObserve LLM cost monitoring](https://openobserve.ai/blog/llm-cost-monitoring/), [anomaly guide](https://openobserve.ai/blog/ai-anomaly-detection-guide/)).
- **Per-span cost ceiling:** alert when any single LLM span exceeds a cost threshold (catches the 50K-token-for-a-3K-token-question case).
- **Hard step caps** per task/run as the deterministic backstop.
- Cloud-side threshold tuning (transferable): fire on **percentage AND absolute floor together** — e.g. ≥20–30% above rolling baseline AND ≥$25 absolute; 5%-above-baseline alerts fire constantly on normal variance; target false-positive rate <10% ([nOps](https://www.nops.io/blog/cloud-cost-anomaly-detection/), [Cast AI](https://cast.ai/blog/kubernetes-cost-anomaly-detection/), [AWS percentage thresholds](https://aws.amazon.com/about-aws/whats-new/2022/12/aws-cost-anomaly-detection-percentage-based-thresholds)).

### The structural timing problem
Cloud billing data (Cost Explorer) lags **up to ~24 hours**, and AWS Budgets evaluates against that delayed data, "so budget actions fire after the money is spent" — while agents spend at API speed ([InfoQ, July 2026](https://www.infoq.com/news/2026/07/ai-agents-billing-guardrails/)). Detection must therefore happen at **action time** (gateway metering, CloudTrail alerts on `InvokeModel`/`RunInstances`), not invoice time.

## 5. Documented runaway-cost incidents and what they teach

**Well-sourced (InfoQ, July 2026** — [article](https://www.infoq.com/news/2026/07/ai-agents-billing-guardrails/)):
- **AWS agency, July 2026:** 3-person firm, normal bill $10–15/mo, hit **$14,000 in one day** after attackers extracted static access keys from an EC2 instance and burned Claude invocations on Bedrock. GenAI credentials are uniquely dangerous: stolen keys convert to dollars almost instantly with no infrastructure to stand up.
- **DN42 hobby-network incident, May 2026:** operator gave an autonomous agent full AWS access to port-scan a hobby network; it provisioned five m8g.12xlarge instances, load balancers, Lambdas, and re-applied its CloudFormation template repeatedly, duplicating the stack. Bill: **$6,531.30** (AWS negotiated to $1,894).
- InfoQ's mitigations: SCPs blocking expensive instance families in agent accounts, CloudTrail action-time alerts, IAM roles over static keys, **scope model access to specific models rather than full-access defaults**, dedicated member accounts per agent workload.

**Weaker sourcing — flag explicitly:**
- The widely-cited "**$47,000 LangChain agent loop**" (two agents in a 4-agent research system exchanging clarification/verification messages for 11 days, discovered only on the invoice; a per-task cap would have killed it under $200) appears only in dev.to posts ([one](https://dev.to/brianrhall/how-to-stop-an-ai-agent-from-burning-47000-in-a-loop-nobody-noticed-3pc9), [two](https://dev.to/utibe_okodi_339fb47a13ef5/the-ai-agent-that-cost-47000-while-everyone-thought-it-was-working-1lg6)) with no named company — treat as plausible illustrative anecdote, not verified fact.
- Smaller reported incidents: a developer burning ~$6,000 in Claude Code credits overnight; a Claude Max subscriber hit with **$1,800 in 2 days** after scheduling overnight Claude Code runs ([devtoolpicks roundup](https://devtoolpicks.com/blog/ai-agents-runaway-claude-code-bills-overnight-2026)) — individually unverified, but consistent with the InfoQ-documented pattern.

**Common lessons across all incidents:**
1. Every incident is *absence of a per-task/per-agent hard cap* + *invoice-time discovery*. Caps at the $100–200 per-task level would have reduced multi-thousand-dollar incidents to noise.
2. The two failure classes differ: **looping agents** (need step caps + budget caps + loop detection) vs **leaked credentials** (need short-lived scoped credentials + action-time alerts). Both are cost incidents; only one is an agent bug.
3. Unattended/overnight operation is the highest-risk window — spend velocity is unchanged but detection latency balloons.

## 6. Spend forecasting

- FinOps Foundation working-group papers exist specifically on this: [Cost Estimation of AI Workloads](https://www.finops.org/wg/cost-estimation-of-ai-workloads/), [How to Forecast AI Services Costs in Cloud](https://www.finops.org/wg/how-to-forecast-ai-services-costs-in-cloud/), [Effect of Optimization on AI Forecasting](https://www.finops.org/wg/effect-of-optimization-on-ai-forecasting/). Claims of forecasts within ~5% of actuals per month exist but assume mature historical baselines; the WGs themselves stress AI budget variance is hard because of thin history and volatile pricing.
- Practical bottom-up formula ([SumatoSoft framework](https://sumatosoft.com/blog/ai-token-cost-calculation-framework-for-forecasting-llm-spend)): users × sessions × avg tokens/session (input and output separately) × per-token price, adjusted for **retry rate and cache-hit rate**, then multiplied by an **agent multiplier of 1× (simple RAG) up to 10× (fully agentic)**. That 10× agentic multiplier is the key structural correction over chat-era forecasting.
- **Concrete anchor benchmarks for "normal" (official Anthropic docs,** [Claude Code costs](https://code.claude.com/docs/en/costs)): enterprise average **~$13 per developer per active day**, **$150–250 per developer per month**, **90% of users below $30/active day**. Agent teams use **~7× more tokens** than standard sessions (each teammate is a separate context window). Cache dynamics matter: a one-line message in a long-lived session still re-reads the whole history; cache lifetime drops from 1h to 5min on usage credits/API.

## 7. Public tooling landscape

| Tool | Layer | Cost-governance capabilities | Pricing (2026) |
|---|---|---|---|
| **Langfuse** | Tracing SDK (self-host or cloud, MIT) | Per-generation token + USD cost on hierarchical traces; filter/aggregate by user, session, metadata; most-adopted OSS LLM-engineering platform | Free hobby; Core ~$29/mo ([docs](https://langfuse.com/docs/observability/features/token-and-cost-tracking), [comparison](https://www.rize.io/ai-tools/vs/langfuse-vs-helicone)) |
| **Helicone** | Gateway/proxy | Automatic cost metrics with no SDK changes; **custom rate limits by request count, cost, or custom property** (per-user, per-org, per-feature caps); property breakdowns for chargeback | Free hobby; Pro ~$79/mo ([docs](https://docs.helicone.ai/features/advanced-usage/custom-rate-limits)) |
| **LiteLLM proxy** | Gateway (OSS) | Virtual keys with per-key/team/user/org/tag budgets, hard caps, `fail_closed_budget_enforcement`; Anthropic-documented enterprise pattern for Claude Code attribution | OSS ([docs](https://docs.litellm.ai/docs/proxy/users)) |
| **OpenMeter** | Usage metering primitive (OSS + cloud) | Token/usage metering for billing and chargeback; the "meter" building block rather than an observability suite | OSS free tier ([comparison](https://futureagi.com/blog/best-llm-cost-tracking-tools-2026)) |
| **AWS Bedrock AIPs** | Cloud billing | Cost-allocation tags per app/tenant → Cost Explorer/CUR; daily granularity, ~24h lag, not retroactive | Included ([AWS docs](https://docs.aws.amazon.com/bedrock/latest/userguide/cost-mgmt-application-inference-profiles.html)) |
| **Provider consoles** (OpenAI, Anthropic) | Provider billing | Org/project (OpenAI) and workspace (Anthropic) hard caps; 429 on breach; alert thresholds | Included ([OpenAI](https://developers.openai.com/api/docs/guides/spend-limits), [Anthropic](https://code.claude.com/docs/en/costs)) |
| **OTel GenAI semconv** | Standard | Vendor-neutral `gen_ai.*` span/token attributes; the portability layer under all of the above | n/a ([Uptrace](https://uptrace.dev/blog/opentelemetry-ai-systems)) |

Pattern in the market: observability (see spend) and enforcement (stop spend) are converging at the **gateway** — Helicone and LiteLLM both do caps + metering in one place; Langfuse sees everything but stops nothing.

## 8. Grounding Greenwood's ~$22k/yr estimate

Greenwood's figure is *governance inference overhead* (message screening cascade) at 500 agents, modeled as ~0.1% of runtime inference (`research/topics/11-scalability-and-cost.md:155-179`). External grounding is indirect — no public source prices message-screening governance specifically — but three checks are possible:

1. **Direction check (supports):** the FinOps consensus is that governance/observability spend must be a small single-digit percentage of the spend it governs; 0.1% is comfortably inside any such bound. Cheapest-first cascade (rules → encoder → small model → LLM only on flagged items) matches the industry's cost-tiering pattern.
2. **Tooling-cost comparison (supports scale):** a full off-the-shelf governance stack for a fleet this size is cheap in absolute terms — Langfuse Core (~$350/yr) or Helicone Pro (~$950/yr) plus provider-native caps ($0). $22k/yr of *inference* for semantic screening is a different (and much larger) line item than tooling; the estimate is defensible only because it buys screening no off-the-shelf tool provides. Record this distinction.
3. **ROI framing (already flagged internally, externally endorsed):** the repo's own open question — does screened messaging save more cascade-redo inference than it costs — mirrors the FinOps Foundation's "measure outcomes per token, not tokens" guidance ([finops.org token economics](https://www.finops.org/insights/token-economics-the-atomic-unit-of-ai-value/)). Instrument redo-tokens-avoided vs governance-tokens-spent from day one.

**Status: the $22k/yr remains an internal model output; external data neither confirms nor refutes it, but its ~0.1%-of-inference framing is consistent with industry norms. Mark as modeled-not-measured wherever it appears.**

---

## Implications for the late-July release (junior engineer, 4–5 apps)

1. **Ship hard per-task and per-agent budget caps as defaults, not options.** Every documented incident is the absence of a bound. Default per-task cap in the $5–20 range and per-agent daily cap sized from a 1–2 week measured baseline; the $47k-class anecdotes all die under $200 with any cap at all. Trellis's per-agent budget caps are the right call — extend the same primitive to every garden component that invokes models.
2. **Fail closed, and treat unpriceable calls as violations.** LiteLLM's bug history shows the two real bypass classes: stale spend counters and unknown-model → $0-cost requests. The garden's enforcement point must validate against authoritative spend and reject calls it cannot price.
3. **Enforce at the gateway, reconcile at the bill.** Attribution metadata (app, agent ID, task/idea ID) must ride on every model call at request time; cloud/provider billing lags ~24h and is day-granularity at best. The Trellis Costs dashboard should read from garden-side metering, with the provider bill as a monthly sanity check.
4. **Graduated cutoff ladder for the junior operator:** alert at 75%, degrade/throttle at 90%, hard-stop at 100% with a clear "budget exhausted — here's how to raise it" message (mirrors Claude Enterprise's 75/90/95% pattern). Hard stop = new work rejected with retryable error, in-flight work completes; a stuck-at-cap agent must be visible on the dashboard, not silently dead.
5. **Runaway detection the junior engineer doesn't have to tune:** (a) deterministic step cap per task; (b) alert when any single call exceeds a per-call cost ceiling; (c) rate alert when an agent's tokens/hour exceeds ~3× its rolling baseline AND an absolute floor (avoid the <5%-threshold alert-fatigue trap); (d) loop signature — repeated similar spans with no state change. Items (a)+(b) are trivially implementable by end of July; (c) needs ~2–3 weeks of baseline data post-launch.
6. **Answer "is this normal?" on the dashboard.** Publish reference baselines next to live numbers: Anthropic's own figures ($13/dev/active-day average, 90% of users <$30/day, $150–250/dev/mo) are usable day-one anchors until per-app baselines accumulate. A junior engineer needs "today: $9 — typical for this app: $6–14" more than a raw dollar figure.
7. **Treat overnight/unattended operation as a special risk class.** Tighter caps (or explicit opt-in budgets) for scheduled/unattended runs; the worst incidents happened while nobody was watching. Agent-team-style fan-out deserves its own multiplier warning (~7× tokens).
8. **Credential hygiene is cost governance.** The single best-sourced incident ($14k/day) was a leaked key, not a loop: short-lived scoped credentials, per-agent workspaces/keys so a leak is capped by that key's budget, and model-scoped access rather than full-access defaults.
9. **Forecast with the agent multiplier and label Greenwood's number honestly.** Use tokens/task × tasks/day × agent multiplier (1–10×) with retry and cache-hit adjustments; expect wide error bars until 30+ days of history. Keep the $22k/yr governance figure marked "modeled, ~0.1% of inference, consistent with but not confirmed by external norms," and wire the redo-tokens-avoided vs governance-tokens-spent counters into the release so the ROI question becomes measurable.
