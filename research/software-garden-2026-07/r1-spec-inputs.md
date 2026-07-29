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

---

# Round 2 (2026-07-29, later the same day) — flagged-gap deep dives

Six dives into gaps the corpus itself flagged. Four complete below; **break-glass and
patrol-trust are pending** (a multi-hour tool-safety-classifier outage blocked agent
launches and web access; see method caveats inside each note — several claims carry
`[verify]` tags for the next web session).

## D. Stat verification (`notes/research-round2-stat-verification.md`)

Verdicts on the 12 headline claims cited in spec discussions:

- **VERIFIED:** Spark-to-Fire ≥89% cascade prevention (actually 89/93/94% by mode vs
  ~2–3% baselines — stronger than quoted); MAST 1,600+ traces, "gains often minimal";
  Anthropic ~15× multi-agent token cost (agents ~4×, multi-agent ~15× chat; tokens
  explain ~80% of BrowseComp variance); METR 19%-slower/believed-20%-faster.
- **CORRECTED:** the "juniors over-approve" framing — arXiv 2602.00496 (N=10,
  qualitative) actually found novices **mis-calibrate in both directions**
  (over-reliance *and* cautious avoidance). Design consequence: approval UX must make
  confidence legible, not merely add friction. Corpus wording needs a correction pass.
  Also: MAST is the UC Berkeley group; Neubig belongs to CAID (CMU).
- **DO NOT CITE (unverifiable):** "Anthropic 47% debugging-skill drop"; the "$47k agent
  loop" anecdote; ">4h agents = 90% higher failure risk". (InfoQ $14k/day and $6.5k
  incidents remain verified.)
- **BLOCKED, must re-verify before the spec hardens:** SREGym's 38.9–72.6% range
  (abstract confirms only "up to 40% end-to-end differences"); arXiv 2606.01435
  (+28-point freshness delta); arXiv 2606.31498 (protocol-gap paper — underpins the
  differentiation thesis; the ID itself flagged as suspicious); CAID +25.6%. Re-run
  list is in the note.

## E. Substrate & runtime (`notes/research-round2-substrate-ops-weight.md`)

Ranked against the "one notch harder than kploy" bar for M1:
1. **Postgres-backed event log behind a Greenwood-owned Kafka-shaped log trait** — zero
   new deployables on bc-prod; P0 mandates event sourcing, not Kafka; all five log
   properties hold at M1 scale (message-db pattern); D6's compaction-keyspace problem
   dissolves (projection = table). Cost: re-opens the decided Kafka-API line; guard =
   CI-test the trait against a real Kafka-API engine from day one.
2. **Redpanda Community 3-node** if the Kafka-API line holds — single C++ binary incl.
   Schema Registry, one notch; caveat: tiered storage is enterprise-licensed (a cost
   feature, not correctness, at M1 — payloads-by-reference on MinIO suffices).
3. **AutoMQ deferred to M2+** as the named D6 endgame (Apache 2.0, solves
   compaction-on-S3, but JVM fork + WAL + controller + S3 chain ≈ two notches — the
   stack shape Chattermax Phase 8 condemned).
- Eliminated: WarpStream (cannot self-host — Confluent-hosted control plane);
  NATS/Pulsar (no compaction semantics); **Temporal/Restate/DBOS betray P0** (private,
  per-workflow, truncated logs — Temporal caps 51,200 events/50MB; Rootlines and D4
  hermetic evals lose their input). Managed Kafka tiers = dev/CI escape hatches only.
- **Runtime language: Rust, decidable now** — rdkafka covers idempotence/transactions,
  Chattermax already shipped an rdkafka idempotent producer, Kafka Streams buys nothing
  for bespoke folds; a thin log trait de-risks the substrate question entirely.
- Method caveat: managed-tier/durable-execution/client sections are training-knowledge
  cross-checked against KB prior research, `[verify]`-tagged; agent asserts no tag
  flips the ranking.

## F. Approval semantics (`notes/research-round2-approval-semantics.md`)

Answers to the four open questions from the round-1 gates dive:
1. **Only human principals approve — no automated identity holds the checker role.**
   Shipped precedent: Entra PIM bars service principals from approving; GitHub required
   reviewers are humans/teams only (bots participate only as registered deterministic
   protection rules). SoD logic: same-garden agents fail independence (shared
   operator/model/compromise surface). Spec: `ApprovalGranted` valid only from
   `Garden::Human`; `checker != maker`; at T4 `checker != originating_principal`.
2. **Cedar entity model:** typed principals (`Garden::Human`, `Garden::Agent`), actions
   auto-generated per tool op (AgentCore convention) each carrying `required_tier`,
   resources = app/environment hierarchy, envelope-verified `originating_principal` in
   context. **Tier law as two forbids:** action tier ≤ min(principal.tier_ceiling,
   originating_principal.max_tier); each denial names its ceiling.
3. **Three TTL clocks.** Precedented: pending-approval hours-to-days (PIM 24h fixed,
   Teleport 1h–1wk); elevation hours (PIM 8h default). No shipped precedent for
   granted-but-unexecuted validity (systems consume approvals instantly) — design
   call: propose T3 pending 4h / validity 15min; T4 pending 24h / validity 60min;
   minted-credential ≤8h; all versioned policy numbers.
4. **Gate placement (CQRS practice, Axon):** Cedar evaluation in a bus-side dispatch
   interceptor; approval invariant in the decider; `evolve` never calls Cedar — the
   interceptor's ruling is recorded as an event stamped `policy_version`, so folds
   replay deterministically under any later policy. (Recorded-decision half is a
   design call grounded in Decider purity; no named external pattern exists.)

## G. Agent identity & signed actions (`notes/research-round2-agent-identity.md`)

**Do not adopt Buzz's keypair-per-agent/signature-per-event for R1** — in a
single-cluster deployment where the operator provisions keys and the bus is the trust
anchor, it's audit theater with real key-management cost (Buzz needs it because its
relay is untrusted). Instead:
- **Reserve three T1.1 envelope fields day one** (identity becomes key management,
  never a migration): `Principal producer_principal {id, kind, runtime}` (id =
  `gen_ai.agent.id`, composing with DC-4); `OriginatingHuman originating_human` with a
  reserved unpopulated `mac` (DC-3); `Signature signature {key_id, scheme, sig}`
  reserved, plus one spec line: **signatures sign the canonical-hash `event_id`**.
- **Impersonation control at the broker, not in envelope crypto:** per-agent Kafka
  principals + ACLs binding principal → producer id (M1–M2).
- **Sign decisions, not thoughts:** if anything is signed early, it's
  `ApprovalGranted`/`PolicyDecision` — highest audit value per unit of key surface.
- SPIFFE/per-agent keypairs only if Greenwood ever federates across trust domains.

## New decision candidates from round 2 (need Terra)

- **DC-6** Substrate for M1: re-open the decided Kafka-API line (Postgres-behind-trait)
  — yes/no; if no, Redpanda Community; AutoMQ stays the M2+ endgame either way (E).
- **DC-7** Runtime language: Rust (E; unblocks P1/P2 under either DC-6 outcome).
- **DC-8** Human-only approvers + tier law as two forbids (F.1/F.2) — extends DC-1/DC-2.
- **DC-9** TTL defaults as versioned policy: T3 4h/15min, T4 24h/60min, credential ≤8h (F.3).
- **DC-10** T1.1 envelope reservations: producer_principal, originating_human.mac,
  signature field + "signatures sign event_id" rule (G) — concretizes DC-3/DC-4.

## Pending round-2 items
- Break-glass design note (agent paused by the outage, self-resuming).
- Patrol-trust / conformance-checking note (launch blocked by the outage, retrying).
- Stat re-verification of the four BLOCKED claims (agent has a retry window armed).
- Corpus correction pass: fix "juniors over-approve" wording per D; strip do-not-cite
  stats.

## Use
Input to the WORK-BREAKDOWN revision (floor/stretch split). Rails tickets should cite
the specific finding (e.g. "T-gate acceptance: PreToolUse veto holds under
bypassPermissions — A.4").
