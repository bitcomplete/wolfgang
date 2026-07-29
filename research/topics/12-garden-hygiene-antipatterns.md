---
doc_type: topic
topic: Garden hygiene anti-patterns — observed failures the greenfield system must not inherit
project: Greenwood
tags: [hygiene, anti-patterns, staleness, defaults, greenfield, release-r1]
sources: [software-garden-2026-07 research sweep]
created: 2026-07-29
updated: 2026-07-29
status: active — checklist input for R1 scope and Greenwood design reviews
summary: Concrete hygiene failures observed in mahdi/trellis/chattermax during the 2026-07 research sweep, each paired with the greenfield countermeasure. Purpose - these are observed, not hypothetical; do not copy them into the new system.
---

# Garden hygiene anti-patterns (observed 2026-07-29)

Source notes: `research/software-garden-2026-07/notes/repo-{mahdi,trellis,chattermax}.md`,
`kb-wolfgang.md`, `research-emergent-1.md`. Each item: **observed failure → greenfield
countermeasure**. Terra's direction (2026-07-29): the new system starts greenfield;
record these so they aren't copied forward.

## 1. Authoritative tone decoupled from data age (mahdi)
Statuses frozen ~7 weeks (stamped 2026-06-09/10) while the navigator persona instructs
"answer authoritatively; only self-doubt when accused of hallucinating." A junior will
take stale confidence as truth.
→ **Countermeasure:** authority is *computed* from verification metadata, never asserted
by persona. Two-date frontmatter (`updated` vs `verified`) on every KB entry; every
answer stamped with data age; confidence degrades on staleness tiers. Deterministic
freshness resolution, not LLM judgment (arXiv 2606.01435: +28 points over LLM
adjudication).

## 2. Stored state that could be derived (mahdi, chattermax README)
mahdi's PROJECT-INDEX claims merge conflicts resolved that git shows unresolved; the
chattermax README describes a system its code has outgrown. Any hand-maintained copy of
derivable state will drift.
→ **Countermeasure:** derive-don't-quote. Anything recomputable (git status, deploy
version, running/stopped, last-event time) is a projection over live signals — in
Greenwood, a fold of the log, fresh by construction. Prose only for what can't be
derived (rationale, intent).

## 3. Push-based freshness protocols go unused (update-mahdi)
mahdi shipped a push update protocol; it received one update-log entry ever. State was
refreshed by occasional manual audits. Freshness that depends on humans/agents
remembering to report in does not happen.
→ **Countermeasure:** freshness must be pull/event-derived (consume the event stream,
CI hooks, deploy webhooks). Where a human note is genuinely required (close-out notes),
make it workflow-enforced — a gate on task completion, not a courtesy.

## 4. Safety machinery shipped with safety defaults off (trellis)
Trellis ships a six-layer defense-in-depth design and defaults to
`sandbox_enabled: false` + `permission_mode: bypassPermissions`. The parts exist; the
defaults invert them. Public best practice (Codex sandbox-mode × approval-policy)
defaults the other way.
→ **Countermeasure:** safe-by-default is a release acceptance criterion, not a config
option. Escalation to less-safe modes is explicit, logged, and time-bounded. Any
default that would fail an incident-review question ("why was the sandbox off?") is a
bug.

## 5. Long-lived unpushed/unmerged state (trellis: 127 unpushed commits; mahdi: 72 dirty entries)
The most mature garden component lived on an unpushed `integration` branch; the
knowledge navigator's own repo carried two unresolved merge conflicts its index claimed
were fixed.
→ **Countermeasure:** the system of record is the remote; a patrol (read-only, T1)
flags unpushed branches / dirty trees / doc-vs-reality drift across the 4-5 apps as a
standing conformance check. (Kasava's expired docs TLS cert is the same failure class —
exactly what garden patrols should catch.)

## 6. In-memory lifecycle state (chattermax freeze/thaw)
Chattermax's freeze/thaw agent lifecycle kept freeze state memory-only; a restart lost
it. The junior's most common operation ("this agent died — resume it") can't sit on
volatile state.
→ **Countermeasure:** P0 already covers this — lifecycle is events, resume is a fold
(WORK-BREAKDOWN P3). Keep it honest at the edges: no component may hold lifecycle state
not derivable from the log (acceptance-test it: kill -9 anything, resume must work).

## 7. The goal missing from the system of record (wolfgang itself)
The July→Aug release goal, persona, and 4-5-apps scenario existed only in conversation
until 2026-07-29; the planning navigator answered planning questions blind to its own
release. Fixed by decisions.md **R1**, but the pattern generalizes.
→ **Countermeasure:** decisions and goals land in the KB the day they're made (this is
CLAUDE.md doctrine — "keep durable design context in the knowledge base instead of
letting it live only in chat"); a navigator that cannot cite a recorded goal for
current work should say so rather than improvise one.

## 8. Unversioned schemas (the original garden event model)
The garden's event formats carried no schema version — the "no-version regret" already
cited in WORK-BREAKDOWN T1.1.
→ **Countermeasure:** already designed in (Schema Registry, `schema_version`-scoped
hashing, `SerializationEpoch`). Listed here so the regret survives as a checklist item
for every *new* persisted format (policy configs, runbook frontmatter, card schemas),
not just the Kafka envelope.

## Use
Treat §1–8 as a review checklist for R1 scope decisions and for every Greenwood design
review: "which of the eight are we about to re-implement?"
