---
doc_type: spec-inputs
topic: R1 spec inputs — consolidated HIGH-relevance findings from the emergent-topic deep dives
project: Greenwood
tags: [r1, release, gates, cost, staleness, spec-inputs]
sources: [research-emergent-1-deep, research-emergent-2-deep, research-emergent-3-deep]
created: 2026-07-29
updated: 2026-07-29
status: active — input to the R1 scope cut and WORK-BREAKDOWN revision; decision candidates pending Terra
summary: Every HIGH-scored finding from the three scored deep dives (staleness, approval gates, cost governance), organized by R1 workstream, with the decision candidates they raise. MEDIUM/LOW findings stay in the notes files.
---

# R1 spec inputs — HIGH-relevance findings (2026-07-29)

Scoring rubric (applied in each deep dive): HIGH = should shape the R1 (2026-08-16)
spec or MVP scope; MEDIUM = shapes M2+; LOW = interesting only. Full scored tables and
citations live in `notes/research-emergent-{1,2,3}-deep.md`. This file collects only
the HIGH set, grouped by the R1 workstream it lands in (floor/stretch split per the
re-ordering proposal under discussion).

## A. Approval gates (R1-Rails floor + one Greenwood design decision)

From `notes/research-emergent-2-deep.md`:

1. **Gate in the fold (Greenwood-native enforcement).** Model approvals as first-class
   events — `ActionProposed → ApprovalGranted → ActionExecuted` — and have the decider
   reject any Execute whose *folded state* lacks a matching, unexpired, parameter-bound
   approval. Non-bypassable by construction, cheap greenfield, unretrofittable later.
   → **Decision candidate for the D-log** (extends P0/D8; needs Terra).
2. **Maker-checker state machine.** Adopt the fintech dual-control aggregate wholesale
   (PENDING/APPROVED/REJECTED/CANCELLED/PROCESSING/FAILED/EXPIRED; self-approval
   blocked; edit-invalidates-approval via target-version capture; idempotent execution
   log; expiry/escalation) as the approval aggregate's schema and invariants.
3. **Cedar as the tier-policy engine.** Represent the T1–T4 matrix as Cedar policies
   evaluated in-process at the tool-call boundary: default-deny, conditions over
   LLM-generated parameters, formally analyzable, no server to run. Industry converged
   here (Bedrock AgentCore Policy GA 2026-03). → **Decision candidate** (vs. OPA /
   hand-rolled rules; interacts with T6.2's "static policy engine").
4. **Bypass-proof enforcement on existing harnesses — shippable today.** Claude Code
   PreToolUse hooks veto tool calls even under `bypassPermissions` /
   `--dangerously-skip-permissions`, and deny rules can't be overridden. This is the
   shipped fix for the trellis inverted-defaults hole (topic 12 §4) and makes the
   R1-Rails floor real without any Greenwood code: wire the hook to the policy check.
5. **Paved-road enforcement on bc-prod.** Agents operate the 4-5 apps only through
   kploy + GitOps + admission gates (Kyverno/OPA) — never raw kubectl credentials.
   Every agent mutation becomes a reviewable commit; existing platform gates apply to
   agents for free. (Also demotes deploys from T4 toward T2 via canary+auto-rollback.)
6. **Originating-human authority in the message envelope.** Carry the initiating
   human's identity/role/auth-strength as an integrity-protected envelope field (OWASP
   ASI03; AWS Cedar delegation model), with the invariant that no delegated action may
   exceed the tier the originating human could authorize directly.
   → **Decision candidate for the D8 envelope schema** (near-free greenfield, painful
   to retrofit).

## B. Cost governance (R1-Rails floor)

From `notes/research-emergent-3-deep.md`:

1. **Per-task caps are config if agents run on the Claude Agent SDK**: `max_turns` +
   `max_budget_usd` per run with `error_max_budget_usd` result subtypes (soft cap,
   checked between turns).
2. **Garden-side enforcement must own the hard stop**: Anthropic spend limits are
   console-only (Rate Limits API is read-only) — provider caps are manual backstops,
   never the enforcement layer.
3. **LiteLLM's tag-budget contract is a copyable spec** for the bus budget primitive:
   `{name, max_budget, budget_duration, soft_budget, models[]}`, HTTP 400
   `budget_exceeded` carrying current/max, mint-time upper bounds on generated keys.
4. **Key-per-agent + Anthropic Usage & Cost Admin API** (1-minute buckets, ~5-min
   freshness, grouped by api_key/workspace/model) = a free near-real-time
   reconciliation layer whose books decompose per-agent.
5. **Deterministic loop rails with production numbers — zero tuning, shippable by
   Aug 16**: identical tool-call fingerprint 2–3× → block; ABAB oscillation detection;
   session rails ~50 turns / 10 min / 500K tokens.
6. **Dual-cap pattern**: per-session $ cap AND $/hour velocity cap (e.g. $50/hr) — the
   anti-overnight-incident control; velocity is a 1-hour tumbling window over the bus's
   own metering events.
7. **Graceful cutoff shape**: at ~90% of budget, inform the agent and have it wrap up
   with a status note; at 100%, halt + persist state + escalate with a trace link.
   "Stopped at limit with a note," never a silently dead process.
8. **The event-sourced bus IS the meter** (OpenMeter proves the architecture:
   CloudEvents → Kafka → exactly-once consumer): attribution, budgets, and velocity
   alerts are just consumers; steal event-id dedup + backfill.
9. **Adopt OTel GenAI semconv attribute names on the greenfield schema**
   (`gen_ai.agent.id`, `gen_ai.conversation.id`, `invoke_agent`/`execute_tool`,
   `gen_ai.client.token.usage`) — free now, buys every OTel dashboard later.
   → **Schema decision candidate** (T1.1 envelope / metering events).
10. **Tool contracts prevent loops**: explicit terminal SUCCESS/FAILED states cut agent
    calls ~7×; make it a spec rule for every garden tool, plus per-tool call limits.
11. **Default per-agent daily cap ~$15–25**, bracketed by Claude Code ~$13/dev-day and
    Devin ~$17/agent-day.

## C. Staleness & operator surface (R1-Rails floor / P11)

From `notes/research-emergent-1-deep.md`:

1. **Projection checkpoint lag is a free, exact staleness metric** — every
   event-sourced view carries "as of offset/event-time, N behind head" with zero
   metadata discipline.
2. **KB staleness monitoring collapses into Kafka consumer-lag monitoring** — Burrow /
   kafka-lag-exporter / stock Grafana time-lag alerts, no new code; mahdi's 7-week
   freeze would have been a red panel on day one.
3. **Critical reads bypass the read model**: operationally critical navigator answers
   check the event stream / live system directly — the event-sourced formalization of
   derive-don't-quote (topic 12 §2).
4. **dbt's `warn_after`/`error_after` + machine-checkable evidence field** is the
   field-tested schema for tiered knowledge TTLs: copy into entry frontmatter; map
   tiers to stamp → banner → refuse.
5. **Runbook staleness discipline**: bind steps to live system facts (dashboard URL +
   embedded query) so drift breaks visibly; `verified` on a procedure means *executed*,
   not re-read; stale steps cost 8–15 min each during incidents.
6. **Derive `updated` from git** (MkDocs/Docusaurus pattern) — only `verified` ever
   needs a human; a stored `updated` field is itself a staleness liability.
7. **Hash-based incremental index sync with stale-record deletion** (LangChain
   RecordManager: none/incremental/full cleanup modes) — any derived index without a
   monitored reconciliation job is a release blocker (the frozen-PROJECT-INDEX fix).

## Decision candidates raised (need Terra — do not treat as decided)

- **DC-1** Gate-in-the-fold: approvals as events required by the decider (A.1).
- **DC-2** Cedar (vs. OPA vs. hand-rolled declarative rules) as the tier-policy engine
  at the tool boundary, shared shape with T6.2 (A.3).
- **DC-3** Originating-human authority field in the D8 message envelope (A.6).
- **DC-4** OTel GenAI semconv names baked into the T1.1 envelope / metering schema (B.9).
- **DC-5** Default numbers as versioned policy: $15–25/agent-day, $50/hr velocity,
  50-turn/10-min/500K-token session rails, 90% wrap-up threshold (B.5/B.6/B.7/B.11).

## Use
Input to the WORK-BREAKDOWN revision (floor/stretch split). Rails tickets should cite
the specific finding (e.g. "T-gate acceptance: PreToolUse veto holds under
bypassPermissions — A.4").
