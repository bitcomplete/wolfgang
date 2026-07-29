---
title: "Cost Governance for Agent Fleets — Deep Dive (pass 2)"
date: 2026-07-29
topic: "Gateway budget implementations, runaway-loop heuristics, event-sourced attribution schemas, graceful-cutoff UX, per-agent spend baselines"
status: draft
---

# Cost Governance Deep Dive (pass 2)

**Builds on:** `research-emergent-3.md` (same date). That note covers the FinOps landscape, provider caps, incident postmortems, forecasting, and tooling table. This pass goes deeper on five gaps: concrete gateway budget mechanics, runaway-detection heuristics with numbers, attribution schemas an event-sourced bus gets for free, what the agent does when the cap hits mid-task, and per-agent (not just per-seat) spend baselines. Scored against the Aug-16 release: junior operator, 4–5 apps, tiny company, greenfield Kafka event-sourced bus (Greenwood).

## Scored findings

| # | Finding | Score | Why |
|---|---|---|---|
| 1 | **LiteLLM's tag-budget schema is a copyable spec**: `{name, max_budget, budget_duration (1s–30d), soft_budget, models[]}`; tags inherit from keys or ride per-request; breach → HTTP 400 `budget_exceeded` with current-cost/max in the message | HIGH | Greenwood shouldn't adopt LiteLLM for Aug 16 (Anthropic-only), but its budget object + error contract is the reference design for the bus's own budget primitive |
| 2 | **Anthropic Usage & Cost Admin API**: usage at 1-minute buckets, ~5-min data freshness, poll 1/min sustained; group/filter by api_key, workspace, model, service_tier; cost endpoint is daily-only | HIGH | The reconciliation layer for the Trellis costs view exists, is near-real-time for usage, and is free; per-API-key grouping = per-agent attribution if each agent gets its own key |
| 3 | **Spend limits are NOT settable via Anthropic's Admin API** — workspace caps are console-only (open feature request); Rate Limits API is read-only | HIGH | Kills any "delegate hard caps to the provider" design: garden-side enforcement must own the hard stop; provider workspace cap is only a manually-set backstop |
| 4 | **Claude Agent SDK ships native cost governance**: `max_turns` + `max_budget_usd` per run, returning `ResultMessage` subtypes `error_max_turns` / `error_max_budget_usd`; budget checked between turns (soft cap, slight overrun possible) | HIGH | If garden agents run on the Agent SDK, per-task caps are a config field, not a build — the single cheapest Aug-16 win |
| 5 | **Deterministic loop-detection heuristics with production numbers**: tool-call fingerprint `hash(tool+inputs)` repeated 2–3× consecutively; ABAB oscillation between two tools; hard rails ~50 turns / 10 min / 500K input tokens per session | HIGH | Zero-tuning, junior-proof, implementable before Aug 16; complements the first pass's baseline-dependent rate alerts |
| 6 | **Dual-cap pattern: per-session $ cap AND $/hour velocity cap** (e.g. $50/hr) — session caps catch long drifts, velocity caps catch fast runaways | HIGH | Two config numbers, no ML; velocity cap is the anti-overnight-incident control the first pass's incidents demanded |
| 7 | **Graceful cutoff = budget-aware wrap-up, not kill**: at ~90% budget, inject budget state and ask the agent to conclude + leave a status note; at 100%, halt, persist state, escalate with trace/replay link | HIGH | Defines the operator UX: "agent stopped at its limit and left a status note" vs. a dead process the junior engineer must forensically debug |
| 8 | **OpenMeter proves the Greenwood metering thesis**: CloudEvents → Kafka → exactly-once consumer → columnar store is the industry's billing-grade metering architecture; dedup via event id + idempotent ingestion | HIGH | The event-sourced bus IS the meter — attribution is a consumer, not a bolt-on; validates building metering as a Kafka consumer rather than buying a proxy |
| 9 | **Attribution field names should follow OTel GenAI semconv**: `gen_ai.agent.id`, `gen_ai.agent.name`, `gen_ai.conversation.id`, span kinds `invoke_agent`/`execute_tool`, metric `gen_ai.client.token.usage` | HIGH | Greenfield schema decision being made right now; adopting standard names is free and buys every OTel-compatible dashboard later |
| 10 | **Tool contracts prevent loops**: explicit terminal states (`SUCCESS: …` / `FAILED: …`) cut agent calls ~7× vs ambiguous responses; per-tool call limits per invocation (e.g. `{search: 2}`) | HIGH | A spec-level rule for every garden tool; costs nothing, removes the largest loop class (tool-controlled retries, 41% in the IAL study from pass 1) |
| 11 | **Per-agent spend anchors**: Devin $500+/mo per agent (~$17/agent-day); all-day frontier coding ≈ $100–200/dev/mo tokens; blended seat+token $200–600/mo; token spend dominates seat cost | HIGH | Sizes the default per-agent daily cap with market data: $15–25/agent-day brackets Devin and Claude Code ($13/dev-day) — defensible defaults for the release |
| 12 | **Gateway market 2026** (LiteLLM = OSS control standard; Portkey = managed granular budgets; Helicone = cost-property rate limits; Cloudflare = edge cache; Kong = enterprise) | MEDIUM | Buy-vs-build matrix for M2 multi-provider; irrelevant while Anthropic-only |
| 13 | **Semantic cycle detection research** (arXiv 2511.10650): embeddings + span-graph analysis; distinguish productive iteration via semantic drift + action diversity; cosine >0.95 over 3 turns as stuck signal | MEDIUM | The ML tier of loop detection — build after deterministic rails + 2–3 weeks of trace data exist |
| 14 | **Ready-made Anthropic admin-API integrations** (Grafana Cloud agentless, Elastic, Datadog, Honeycomb, CloudZero, Vantage) | MEDIUM | bc-prod already runs Grafana; polling the admin API into it is a near-zero-cost reconciliation dashboard — but garden-side metering stays primary |
| 15 | **Outcome-based pricing is the 2026 unit-economics frontier**: Agentforce $2/conversation or $0.10/action; Sierra charges per successful resolution; "cost per commit" now an Anthropic-reported metric | MEDIUM | Points the Trellis dashboard's M2 metric at cost-per-task-completed, not tokens; not needed to ship |
| 16 | **Single-vendor governance is a silo** (Finout): provider dashboards can't see cross-provider shift; governance must live above the provider — allocation, unit economics, cross-provider | MEDIUM | Justifies garden-side metering as the system of record even while single-provider; becomes operative at M2 multi-provider |
| 17 | **Framework turn-cap norms**: LangGraph `recursion_limit` (default ~25), OpenAI Agents SDK `max_turns` → `MaxTurnsExceeded`; every major harness ships a hard turn cap | MEDIUM | Confirms bus-level max-turns as table stakes; specific values are framework trivia |
| 18 | Claude Agent SDK gap: no `max_total_tokens` cap (only USD + turns), open feature request | LOW | Edge case; USD cap covers the risk |
| 19 | Anthropic cost endpoint excludes Priority Tier; Workbench usage has null api_key; default workspace = null workspace_id | LOW | Reconciliation footnotes for whoever builds the poller |
| 20 | Sierra $150M+ ARR on per-resolution pricing; Agentforce Flex Credits mechanics ($500/100K credits, 20 credits/action) | LOW | Market color only |

---

## 1. Gateway/proxy budget mechanics in practice

### 1.1 LiteLLM as the reference budget implementation

The first pass covered LiteLLM's failure modes; this pass extracts its **budget object model**, which is the most complete public spec of agent-fleet budgeting and worth copying into Greenwood's own primitive:

- **Budget entities**: per virtual key, per team, per user, per org, and — most relevant — per **tag** ([tag budgets](https://docs.litellm.ai/docs/proxy/tag_budgets), [budgets & rate limits](https://docs.litellm.ai/docs/proxy/users)).
- **Tag schema** (`/tag/new`): `name` (unique id), `max_budget` (USD float), `budget_duration` (`1s`–`30d`; resets at `budget_reset_at`), `soft_budget` (warning threshold), `models[]` (restrict which models the tag may use).
- **Tag attachment**: attach tags to a key at `/key/generate` so *every request with that key inherits the tag and its budget with no client cooperation* — or pass per-request via `metadata.tags` / `x-litellm-tags` header. Multiple tags per request → multi-dimensional attribution (app + agent + task on one call).
- **Breach contract**: HTTP 400, `type: "budget_exceeded"`, message includes tag, current cost, and max budget (`"Budget has been exceeded! Tag=engineering Current cost: 505.50, Max budget: 500.0"`). Requests are rejected until reset or raise.
- **Key-creation guardrails**: `upperbound_key_generate_params` caps what any generated key may be granted (max_budget, duration, tpm/rpm) — i.e., the system that mints agent credentials cannot mint an unbounded one. Directly transferable: **Greenwood's agent-provisioning path should enforce an upper bound on any budget it grants.**
- Also relevant: [provider budget routing](https://docs.litellm.ai/docs/proxy/provider_budget_routing) (per-provider spend ceilings with automatic rerouting) — M2 material.

**Design takeaway for Aug 16:** don't deploy LiteLLM (single provider, one more moving part for a junior operator); *reimplement its contract* — budget = `{scope, max_usd, window, soft_threshold, allowed_models}`, breach = typed error carrying current/max, mint-time upper bounds.

### 1.2 Anthropic's provider-side surface: rich reads, no programmatic writes

- The **Usage & Cost Admin API** ([docs](https://platform.claude.com/docs/en/manage-claude/usage-cost-api)) gives: usage (`/v1/organizations/usage_report/messages`) in `1m`/`1h`/`1d` buckets (1m up to 1,440 buckets), grouped/filtered by **api_key_id, workspace_id, model, service_tier, context_window**; token detail split into uncached input / cached input / cache creation / output. **Data appears within ~5 minutes** of request completion; sustained polling at 1/min is supported. Cost (`/v1/organizations/cost_report`) is **daily-only**, USD in cents, grouped by workspace or description.
  - Practical consequence: a **key-per-agent** (or key-per-app) issuance policy makes the provider's own books attribute spend per agent with zero garden code — the reconciliation view aligns with garden-side attribution by construction.
- The **Rate Limits API is read-only** ([docs](https://platform.claude.com/docs/en/manage-claude/rate-limits-api)), and there is **no Admin API endpoint to set workspace spend/rate limits** — an open feature request ([anthropics/claude-quickstarts#371](https://github.com/anthropics/claude-quickstarts/issues/371)). Finout's analysis of the July 2026 release reaches the same conclusion: dashboards, model-access controls, and 75/90% alerts arrived, but data is fragmented across endpoints and true control loops still have to be built above the provider ([Finout](https://www.finout.io/blog/anthropic-keeps-signaling-where-ai-cost-governance-needs-to-go.-its-not-all-the-way-there-yet)).
  - **Consequence for the release spec:** the provider workspace cap is a *manually configured backstop* (set once in the console); the *operative* hard caps live in the garden. No design should assume the garden can adjust provider limits at runtime.
- Turnkey pollers exist: **Grafana Cloud** has an agentless Anthropic integration with prebuilt dashboards/alerts; Elastic, Datadog, Honeycomb, CloudZero, Vantage likewise ([partner list in the same doc](https://platform.claude.com/docs/en/manage-claude/usage-cost-api), [Elastic](https://www.elastic.co/observability-labs/blog/anthropic-claude-api-monitoring)). Since bc-prod already runs Grafana, this is a cheap day-one reconciliation dashboard.

### 1.3 The 2026 gateway market (for M2)

Consensus across comparisons ([Deepak Gupta](https://guptadeepak.com/tools/top-5-ai-gateways-2026/), [Braintrust](https://www.braintrust.dev/articles/ai-gateway-comparison-2026), [Lushbinary](https://lushbinary.com/blog/ai-gateway-llm-routing-comparison-litellm-portkey-cloudflare/)): LiteLLM = OSS standard for control/coverage; Portkey = managed, most granular budget+rate limits; Helicone = cost/property-based custom rate limits; Cloudflare AI Gateway = near-zero-setup edge caching + analytics; Kong = for existing Kong shops. Common framing: *once real money flows, the gateway is where cost, reliability, and governance are actually enforced*. For Greenwood the "gateway" role is played by the bus's model-call service — same chokepoint, same responsibilities.

## 2. Runaway-loop detection heuristics (with numbers)

Three tiers, cheapest first. Tier 1 needs no data; tier 2 needs config; tier 3 needs traces + embeddings.

### 2.1 Deterministic rails (ship Aug 16)

From practitioner harness guides ([Rahul Kashyap, harness observability](https://rahulkashyap.dev/blog/harness-observability.html), [AWS on Strands hooks](https://dev.to/aws/how-to-prevent-ai-agent-reasoning-loops-from-wasting-tokens-2652), [BSWEN](https://docs.bswen.com/blog/2026-03-11-prevent-ai-agent-infinite-loops/)):

- **Tool-call fingerprinting**: `hash(tool_name + sorted(inputs))` over a sliding window of ~3–5 calls; **same fingerprint 2–3× consecutively (especially with same output hash) = stuck** — block the call before execution (the AWS pattern cancels the tool via a before-tool-call hook, ~30 lines).
- **Oscillation detection**: alternating ABAB between two tools for K cycles → force exit.
- **Per-tool call limits per task**: e.g. `{search_flights: 2}` — a hard per-tool ceiling that resets each invocation.
- **Session hard rails** (example production values): **max ~50 turns, max ~10 minutes wall clock, max ~500K input tokens** per session.
- **Framework norms confirm turn caps as table stakes**: LangGraph `recursion_limit` (default ~25, raised via config; runaway-to-limit is a known bug class — [langgraph#6731](https://github.com/langchain-ai/langgraph/issues/6731)); OpenAI Agents SDK `max_turns` raising `MaxTurnsExceeded` ([docs](https://openai.github.io/openai-agents-python/running_agents/)); Claude Agent SDK `max_turns` (§4.1).

### 2.2 Velocity + statistical (config now, tune with data)

- **Dual caps**: a **per-session spend cap** AND a **$/hour velocity cap** (worked example: $50/hr). "A session cap catches long sessions. A velocity cap catches fast runaway loops before they do serious damage" ([Kashyap](https://rahulkashyap.dev/blog/harness-observability.html)). Velocity is computable from the bus's own metering events with a 1-hour tumbling window — natural Kafka material.
- **Output-similarity**: cosine similarity **>0.95 (or exact hash match) for 3+ consecutive turns** = reasoning loop without progress.
- **Context drift**: increasingly generic responses / vague tool arguments as a soft "lost the plot" indicator.
- The first pass's IQR-over-21-day-baseline and %-plus-absolute-floor alerting slots in here once history accumulates.

### 2.3 Research tier (M2+)

"Unsupervised Cycle Detection in Agentic Applications" ([arXiv 2511.10650](https://arxiv.org/pdf/2511.10650)): embeddings (text-embedding-3-large / Qwen3-Embedding) over span trees; cycles = revisiting semantically-equivalent states even when surface details differ. Key discriminators between *productive* iteration and *stuck* loops: **semantic drift between iterations** (legitimate loops change meaning), **action diversity** (progress varies action types), and DAG-shape differences. Unsupervised — thresholds derived from the traces themselves. This is the eventual "smart" layer over Greenwood's span events; not Aug-16 work.

### 2.4 Prevention beats detection: tool-contract design

The AWS write-up's strongest result is not a detector: giving tools **explicit terminal states** (`"SUCCESS: Booking HT79265 confirmed"` vs. ambiguous output) cut an agent's calls from 14 to 2 (~7×) ([dev.to/aws](https://dev.to/aws/how-to-prevent-ai-agent-reasoning-loops-from-wasting-tokens-2652)). Combined with pass 1's finding that tool-controlled retries cause 41% of infinite loops, this makes **"every garden tool response must declare SUCCESS/FAILED terminally"** a spec-level rule — the cheapest cost control in the whole program.

## 3. Attribution schemas: what the event-sourced bus gives for free

### 3.1 OpenMeter validates the architecture

OpenMeter (now Kong-owned) is the industry's open usage-metering engine, and its architecture is *exactly* Greenwood's shape: **CloudEvents-format events → Kafka topics → custom Go consumer with validation + consistent deduplication + exactly-once inserts → ClickHouse** for aggregation ([how it works](https://openmeter.io/docs/metering/events/how-it-works), [ClickHouse post](https://openmeter.io/blog/how-openmeter-uses-clickhouse-for-usage-metering), [consistent Kafka consumer](https://openmeter.io/blog/consistent-kafka-consumer), [dedup challenges](https://openmeter.io/blog/usage-deduplication)). Lessons to steal rather than rediscover:

- **Idempotent ingestion via event id** — retried model-call events must not double-count spend; dedup is called out as the hard part of billing-grade metering.
- **Exactly-once into the aggregate store** — at-least-once + dedup at the consumer, not hope.
- **Backfill/correction APIs** — pricing errors (see pass 1's unknown-model→$0 bypass) need retroactive repair of aggregates.
- Metering is **a consumer of the event log**, not a proxy in the request path — which means Greenwood's cost dashboard, budget counters, and velocity alerts are all just Kafka consumers over events the bus already records. This is the concrete sense in which the event-sourced design gets attribution "for free": the only new requirement is that **every model-call event carries the attribution envelope at emit time** (below), because events are immutable and tags are not retroactive (same lesson as Bedrock's non-retroactive tags in pass 1).

### 3.2 Field naming: adopt OTel GenAI semconv

The OTel GenAI registry (still marked Development as of semconv 1.40, but the only standard in town) defines the names ([registry](https://opentelemetry.io/docs/specs/semconv/registry/attributes/gen-ai/), [cheat sheet](https://techbytes.app/posts/opentelemetry-genai-agent-semconv-cheat-sheet-2026/)):

- `gen_ai.agent.id` / `gen_ai.agent.name` / `gen_ai.agent.version` — the agent identity.
- `gen_ai.conversation.id` — correlate messages in one conversation/session.
- Span kinds: `create_agent`, `invoke_agent`, `execute_tool`; required duration metric `gen_ai.client.operation.duration`; recommended `gen_ai.client.token.usage`.

**Minimal attribution envelope for every Greenwood model-call event** (synthesis — mark as proposal, not sourced): `{event_id (dedup key), app_id, gen_ai.agent.id, task_id (per-idea/work-item), gen_ai.conversation.id, model, service_tier, tokens{uncached_in, cached_in, cache_create, out}, cost_usd (priced at emit; error if unpriceable), turn_index, tool_name?}`. `app_id` and `task_id` are garden-specific extensions; everything else keeps standard names so any OTel backend can consume the stream later.

### 3.3 Provider-side alignment

Because Anthropic's usage API groups by `api_key_id` and `workspace_id` (§1.2), issuing **one API key per agent** (or minimally per app) makes provider billing decompose along the same axes as bus attribution — monthly reconciliation becomes a join, not an allocation exercise. This also caps blast radius per pass 1's credential-leak finding: a leaked per-agent key is bounded by that key's workspace cap.

## 4. Graceful cutoff: what the agent does when the cap hits mid-task

### 4.1 SDK-native semantics (the floor)

The Claude Agent SDK already implements per-run caps: `max_turns` (counts tool-use turns) and `max_budget_usd`; on breach the run ends with a `ResultMessage` whose subtype is `error_max_turns` or `error_max_budget_usd`. The budget check runs **between turns, not mid-generation** — a soft cap that can slightly overshoot ([agent loop docs](https://code.claude.com/docs/en/agent-sdk/agent-loop), [Promptfoo provider notes](https://www.promptfoo.dev/docs/providers/claude-agent-sdk/)). Gap: no token-denominated cap yet ([claude-agent-sdk-python#1024](https://github.com/anthropics/claude-agent-sdk-python/issues/1024)). For Aug 16, if garden agents run on this SDK, per-task caps are configuration — Greenwood's job is choosing defaults and handling the error subtype visibly.

### 4.2 The wrap-up pattern (the UX)

Practitioner consensus ([harness architecture deep dive](https://dev.to/alexmercedcoder/designing-your-own-ai-harness-a-deep-dive-into-the-architecture-of-agent-loops-tools-context-2knl), [Zylos degradation patterns](https://zylos.ai/research/2026-02-20-graceful-degradation-ai-agent-systems/), [Augment on async workflows](https://www.augmentcode.com/guides/async-ai-agent-workflows)):

1. **Budget-aware wrap-up**: near the cap (~90%), *tell the agent its budget state and instruct it to conclude and summarize progress* rather than killing it mid-thought — converting "runaway agent burned $200 overnight" into "agent stopped at its limit and left a status note." This costs one extra cheap turn and is the single highest-leverage UX decision for a junior operator.
2. **Circuit-breaker runbook at 100%**: halt execution, **dump/persist session state**, log the tripping cause, escalate via webhook/Slack **with a session-replay/trace link** — so the human debugs instead of blindly retrying ([Kashyap](https://rahulkashyap.dev/blog/harness-observability.html)).
3. **Checkpoint-and-resume**: design agents assuming the process dies mid-task; resume from last checkpoint, not from scratch. Persistent checkpointing decouples submission from completion so work survives budget stops the same way it survives crashes and approvals ([Augment](https://www.augmentcode.com/guides/async-ai-agent-workflows), [ResearchGym](https://arxiv.org/pdf/2602.15112) treats budget enforcement as one of the routine interruptions 12–24h runs must survive). On an event-sourced bus, the checkpoint is largely free — the task's event history *is* the resumable state; "resume after raise" = re-dispatch with prior context.
4. **Surface the degraded state immediately**: the operator UI must show "paused: budget exhausted — [raise] [abandon] [view status note]" faster than logs reveal root cause; a stuck-at-cap agent must never look identical to a working one ([Xcapit UX](https://www.xcapit.com/en/blog/designing-ux-ai-agents), pass 1's Trellis implication #4).
5. Error contract: LiteLLM's 400 `budget_exceeded` with current/max in-message (§1.1) is the right shape — machine-typed, human-readable, names the scope that tripped.

**Open design point pass 1 flagged, now sharper:** nobody in the market auto-resumes paused queues when a budget is raised or resets. Greenwood can do this cheaply (budget-raise event → re-dispatch consumer) and it is genuinely differentiating — but for Aug 16, manual "raise + resume" buttons suffice.

## 5. Per-agent / per-seat spend baselines, 2025–2026

Extends pass 1's Claude Code anchors ($13/dev/active-day; $150–250/dev/mo; 90% < $30/day) with per-agent and market-wide numbers ([GetDX pricing/ROI guide](https://getdx.com/blog/ai-coding-assistant-pricing/), [Morph token math](https://www.morphllm.com/ai-coding-costs), [SSOJet comparison](https://ssojet.com/blog/ai-coding-agents-compared)):

- **Devin: $500+/mo per autonomous agent** — the clearest public "price of one agent seat," ≈ $17/agent-day. Together with Claude Code's $13/dev-day this brackets a defensible **default per-agent daily cap of ~$15–25** for garden agents, tightened per-app once baselines accumulate.
- Heavy subscription users migrate to **$100–200/mo tiers**; all-day frontier-model agentic use runs **$100–200/dev/mo in raw token spend**; blended seat+token for teams mixing inline+agentic tools: **$200–600/dev/mo**. Max-20x at full utilization ≈ $200/mo vs $600–1,500/mo API-equivalent.
- **Token spend, not seat licenses, is the dominant cost line**; agents that retry less and cache aggressively pay back fastest — reinforcing §2.4 (tool contracts) and pass 1's cache-dynamics point as *cost* features, not quality features.
- **Outcome-denominated pricing is emerging as the unit-economics frontier**: Salesforce Agentforce at **$2/conversation** or Flex Credits ≈ **$0.10/standard action** ([Agentforce pricing](https://www.eesel.ai/blog/agentforce-pricing), [SaaStr](https://www.saastr.com/salesforce-now-has-3-pricing-models-for-agentforce-and-maybe-right-now-thats-the-way-to-do-it)); Sierra charges **per successful resolution** and crossed $150M+ ARR on it ([witn](https://www.thewitn.com/blog/ai-agent-pricing-in-2026-what-fin-sierra-decagon-and-agentforce-actually-charge)); Anthropic now reports **cost-per-commit** beside spend ([Finout](https://www.finout.io/blog/anthropic-keeps-signaling-where-ai-cost-governance-needs-to-go.-its-not-all-the-way-there-yet)). For Trellis M2: the operator metric should trend toward **$/task-completed per app**, with $0.10/action and $2/conversation as external sanity anchors for "is our per-task cost insane?"

## 6. Cross-provider governance (why garden-side metering is the system of record)

Finout's critique of the July 2026 Anthropic release generalizes: per-vendor dashboards produce "well-governed silos"; enterprises average 3–5 AI providers, and only a layer *above* providers can do allocation, unit economics, and cross-provider comparisons ([Finout](https://www.finout.io/blog/anthropic-keeps-signaling-where-ai-cost-governance-needs-to-go.-its-not-all-the-way-there-yet)). Greenwood is single-provider today, but the bus-side metering layer (§3) is exactly this "above the provider" position — the design holds when a second provider arrives at M2 with zero attribution rework. Record as rationale on the metering decision.

---

## Consolidated additions to the Aug-16 spec (beyond pass 1's nine implications)

1. **Budget primitive spec** = LiteLLM's contract: `{scope, max_usd, window, soft_threshold, allowed_models}`; typed `budget_exceeded` error with current/max; mint-time upper bounds on any budget the provisioning path can grant (§1.1).
2. **Enforcement topology**: garden owns hard caps (provider limits are console-only backstops, not programmable) (§1.2); one Anthropic API key per agent so provider books reconcile per-agent (§3.3).
3. **Loop rails day one**: tool-call fingerprint dedup (block on 2–3 repeats), per-tool call limits, ~50-turn/10-min session rails, session $ cap + $/hr velocity cap; SDK `max_turns`+`max_budget_usd` on every run (§2.1–2.2, §4.1).
4. **Tool spec rule**: every tool returns explicit terminal SUCCESS/FAILED states (§2.4).
5. **Event schema**: attribution envelope with OTel gen_ai names + `app_id`/`task_id` extensions, event-id dedup, price-at-emit with fail-on-unpriceable (§3.2).
6. **Cutoff UX**: 90% → budget-aware wrap-up prompt; 100% → halt, persist, status note, dashboard "paused: budget exhausted" with raise/abandon actions and trace link; manual resume for v1, event-driven auto-resume as a noted M2 differentiator (§4).
7. **Default cap sizing**: ~$15–25/agent-day defaults, anchored to Devin ($17/agent-day) and Claude Code ($13/dev-day) (§5).
