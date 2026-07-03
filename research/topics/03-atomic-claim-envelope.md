---
doc_type: topic-note
topic: Event envelope / message granularity — atomic claims as first-class units
project: Greenwood
tags: [event-envelope, message-schema, atomic-claims, granularity, versioning, three-vocabularies, chattermax, gardener, bitmap]
sources: [spark-to-fire]
created: 2026-06-30
updated: 2026-06-30
status: active
summary: The garden's message IS the atom — NO sub-message/atomic-claim concept exists. It has 3 unreconciled event vocabularies (Chattermax XMPP <chibi>, Gardener JSONL journal, Bitmap BIT-8) and a PLANNED-but-undefined "shared envelope". No version field, no migration plan. Greenwood's atomic-claim envelope (paper's unit of governance) is genuinely new.
---

# Atomic-claim envelope

## Why this matters
The paper governs at the **atomic claim** level (decompose each message into minimal
declarative claims `{aₖ}`, screen each independently). Branch-pruning needs this
granularity — you prune a faulty *claim* and its dependents, not a whole message.

## Garden state  [high-confidence]
- **The message is the atom. NO atomic-claim / sub-message concept exists**
  (verified negative search: no "atomic claim"/"claim"-as-unit anywhere).
- Only sub-structure that exists is *internal to a message*: typed payload fields
  (`files[]`, `conflicts[]`, `location{}`, `metadata{}`) and **stream chunking**
  (`stream_id`/`sequence`/`complete`) — chunking, not claims.
- Chattermax's envelope = XMPP `<message>` wrapping a `<chibi xmlns="urn:xmpp:chibi:0">`
  element with `<type>`, `<context>`, `<metadata>` k/v. 12 types (see topic 00 / the
  full field schemas are in the prior system's message-type spec).

## Three unreconciled event vocabularies  [prior-art]
1. **Chattermax** — XMPP `<chibi>` types → Kafka → S3 (intended canonical spine).
2. **Gardener journal** — local append-only JSONL; 10 domain events: `SeedPlanted`,
   `KnowledgeAdded`, `InteractionRecorded`, `PipelineSucceeded`, `PipelineFailed`,
   `SleepCycleStarted`, `OPLoRAComputed`, `SeedPromoted`, **`SeedRewound`**,
   **`SeedForked`**. (Note the fork/rewind events — see topic 04.) Per-entry field
   schema NOT documented.
3. **Bitmap BIT-8** — "unified event model" + GitHub PR-feedback sense; experimental
   local bridge, no envelope discipline.
- A **"shared event envelope"** to unify these is a an open action item but is
  **undefined (no fields) and stalled** — blocked on prior-system work
  and Gardener T-tier (not started). The "third event vocabulary" note is essentially
  a fragmentation warning.

## Versioning / migration  [prior-art]
- **No `version` field in any schema.** Schema evolution is an OPEN QUESTION, not a
  solved design. Documented open concerns: schema evolution / breaking changes,
  replay performance, snapshot strategy, retention/GC, partial publishing during
  dual-track.

## Implications for Greenwood
- **Design the envelope around atomic claims from day one.** A message carries N
  atomic claims; each claim is the node in the lineage DAG (topic 01) and the unit of
  screening/pruning. This is the single biggest structural departure from Chattermax.
- Claim-level fields likely needed: `claim_id`, `kind` (factuality|faithfulness),
  `derived_from[]`, `supports[]`/`contradicts[]`, `evidence_ref` (the bundle it was
  conditioned on — paper's `Eᵢ(t)`), `confirmed|unverified|rejected` state,
  `source_agent`, `timestamp`, `risk/centrality` annotations.
- **Bake in versioning now** (envelope `schema_version`, migration story) — the
  garden's explicit regret was treating schema evolution as a deferred open question.
- **Unify the vocabularies up front** rather than accreting three — Greenwood is
  greenfield, so define one envelope that the lineage DAG, governance, and lifecycle
  (fork/rewind) all speak. Learn from the garden's fragmentation, don't repeat it.
- Tension: atomic-claim decomposition needs an LLM pass (paper uses FActScore-style
  decomposition + NLI for entailment). That's a per-message cost — budget it, and
  consider doing it lazily / only for hub-role or high-risk messages (paper's
  Balanced policy).

## Open questions
- ~~Who decomposes claims — the producing agent (self-declared) or a bus-side
  decomposition service?~~ **Decided (D1 2026-06-30, revised D7 2026-07-02):** bus-side
  decomposer + mechanical runtime provenance (ContextManifest) + classifier triage;
  agent-declared edges accepted only as untrusted hints. Self-declaration lost — the
  audited party must not author the graph the auditor samples from.
- Can coarse messages and atomic claims coexist (claims as an optional refinement
  layer) for cheap paths? (Partly answered by D7's triage: every claim gets the cheap
  classifier pass; only yellow/hub/boundary claims get the expensive one.)
