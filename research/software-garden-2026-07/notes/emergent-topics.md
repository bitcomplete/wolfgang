# Emergent research topics for the late-July release

Proposed by the emergent-topics agent from the discovery findings (mahdi, trellis,
chattermax, wolfgang KB, Buzz, Kasava, Efecto, Bit Complete). Date: 2026-07-29.

Constraint honored: these do NOT duplicate the already-planned topics (multi-agent
SDLC orchestration, practitioner hard experiences, cognitive debt, standardizing
across many apps, agents for planning/slicing, human-agent interfaces).

## Topic 1 — Keeping agent-facing knowledge fresh: staleness detection and authority calibration

**Question.** What public techniques, tools, and research exist for (a) detecting and
scoring staleness/drift in documentation and knowledge bases that agents answer from,
and (b) making an AI system's confidence/authority proportional to the age and
verification status of its underlying data — e.g., docs-as-code drift detection,
freshness metadata and last-verified stamps, derived-vs-stored state for docs, and
recency-aware retrieval/grounding?

**Why it emerged.** mahdi is the garden's live counterexample: statuses frozen ~7
weeks (2026-06-09/10), 72 dirty git entries, two unresolved merge conflicts that
PROJECT-INDEX claims are resolved, while its persona instructs it to answer
authoritatively and only self-doubt when accused of hallucinating. The push-based
update-mahdi protocol was under-exercised (one update-log entry; state refreshed by
manual audits). Chattermax's README being badly stale versus its code is the same
disease in a different organ.

**Release relevance.** The July release puts a junior engineer in front of
navigator-style answers about 4-5 running apps. A junior will not push back on a
confident wrong answer. The release spec needs automated freshness (derive state from
git/deploy/runtime signals, stamp answers with data age, decay authority) rather than
relying on agents to report in — and this is a well-researched public space (doc
drift tooling, RAG grounding with recency, KB hygiene literature) we have not yet
mined.

**Sources in discovery:** mahdi findings 7, 8, 11, 12; chattermax finding 9;
wolfgang KB grounding rules (CLAUDE.md).

## Topic 2 — Risk-tiered approval gates and least-privilege authorization for agents operating production systems

**Question.** What established frameworks, industry patterns, and research exist for
deciding which autonomous-agent actions on production systems require human approval
versus proceeding automatically — risk/blast-radius tiering, change-management
policy (ITIL/SRE change classes), least-privilege agent credentials and credential
brokering, sandbox-by-default execution, and audit/attestation of agent actions —
and how are agent platforms (and adjacent fields like ChatOps and CD approval
workflows) implementing them today?

**Why it emerged.** Three independent signals: (1) trellis ships a six-layer security
model but sandbox_enabled defaults to false and agents run with bypassPermissions;
(2) both competitors punt exactly here — Buzz has no approval gates or fine-grained
agent permissions, Efecto has no governance/audit story — making safety rails the
garden's clearest differentiation; (3) mahdi's agentic-engineering-insights doctrine
already sketches risk tiering by blast radius as "a ready-made policy skeleton for
what a junior may approve alone versus escalate" but it is unvalidated against public
practice. Greenwood's D8 per-message governance cascade needs the same grounding.

**Release relevance.** The first user is a junior engineer supervising agents that
touch 4-5 production apps. The single most consequential spec decision is the
approval-policy matrix: what the agent may do silently, what the junior may approve
alone, what escalates. Getting this wrong is either an outage (too permissive) or a
useless product (everything escalates). Public research — SRE change management,
production-access literature, agent-permission frameworks, credential brokers —
directly informs it before code freezes.

**Distinct from planned topics.** This is authorization *policy and mechanism*, not
human-agent interface UX and not orchestration.

**Sources in discovery:** trellis findings 6, 12; Buzz findings 6, 9; Efecto findings
9, 10; mahdi finding 10; wolfgang KB findings 4, 12; Bit Complete summary
("governance/screening ... MVP-critical").

## Topic 3 — Cost governance for agent fleets: budgets, runaway detection, and spend attribution

**Question.** What is the state of the art in FinOps for LLM/agent workloads —
per-task and per-agent cost attribution, hard budget caps and graceful cutoff
behavior, anomaly detection for runaway agent loops, forecasting agent spend, and the
public tooling landscape (Langfuse, Helicone, OpenMeter, cloud-provider AI cost
tools) — and what real-world runaway-agent cost incidents teach about defaults?

**Why it emerged.** Trellis unexpectedly treats cost as a first-class *operator*
concern (per-idea dollar figures, a Costs dashboard, cost_usd per transcript,
per-agent budget caps in registry.yaml) — none of the discovery inputs predicted
that, and no other garden component matches it. Meanwhile the Greenwood KB carries a
~$22k/yr governance-overhead estimate at 500 agents that is currently un-grounded in
external data, and Kasava's published AI-credit metering formula shows users expect
transparent unit economics.

**Release relevance.** A junior engineer cannot judge whether $40/day of agent spend
on app #3 is normal or a looping agent burning money; a bootstrapped consultancy
running the garden as internal IP will feel every runaway directly. The release needs
opinionated defaults — caps, alerts, attribution — chosen from evidence rather than
invented, and this is fully answerable from public FinOps/LLM-observability material
and incident write-ups.

**Sources in discovery:** trellis findings 9 (cost first-class) and 6 (budget caps);
wolfgang KB finding 4 ($22k/yr estimate); Kasava finding 5 (credit formula);
Bit Complete summary (bootstrapped, profitable — cost-sensitive).

## Honorable mentions (considered, not selected)

- **Operational weight of Kafka for a tiny team** (AutoMQ/Redpanda/NATS alternatives)
  — real, but it is already a recorded open design question (D6) in the wolfgang KB
  with Terra engaged, so it is less "emergent" and partly an internal decision.
- **Agent-assisted runbooks / copy-paste diagnostic prompts as documentation**
  (trellis troubleshooting.md, chattermax playbooks) — strong pattern, but it borders
  the planned "human-agent interfaces" and "standardizing across many apps" topics.
- **Per-agent cryptographic identity** (Buzz's Nostr keypairs, Sigstore in trellis) —
  folded into Topic 2's audit/attestation angle rather than standing alone.
