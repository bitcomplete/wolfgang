---
doc_type: topic-note
topic: Provenance/lineage DAG vs. flat temporal event stream
project: Greenwood
tags: [event-sourcing, provenance, lineage-graph, causality, correlation-ids, projection, chattermax]
sources: [spark-to-fire]
created: 2026-06-30
updated: 2026-06-30
status: active
summary: The garden's event model is TEMPORAL (chronological replay, frozen context = pointer to event RANGE), with only implicit pairwise correlation IDs and NO first-class causal/lineage DAG. Greenwood must ADD the genealogy DAG as a projection over the log — the substrate supports it; the structure is absent.
---

# Provenance vs. flat event stream

## The central tension (Terra's driver)
Greenwood must be "more than an event-stream — preserve the tree-structure of
agentic interactions so we can monitor and prune faulty branches." The *Spark to
Fire* paper formalizes the target structure: a **Lineage Graph** of atomic claims
with `supports`/`contradicts` edges and confirmed/unverified state (see paper review).

## What the garden HAS (documented)  [high-confidence]
- **Temporal/chronological model only.** "State is reconstructed by replaying events
  [in time order]… Frozen contexts are pointers to event RANGES, not disk snapshots…
  Time-travel debugging replays events from cold storage." [architectural-principles.md]
- Event log "Queryable by room, agent, timestamp" — indexed on temporal/identity
  axes, NOT causal ones. [chattermax/core-architecture.md]
- **Implicit pairwise provenance** via correlation-ID fields in the 12 message types:
  - `tool_result.call_id` → originating `tool_call.call_id`
  - `integration.integrating` → the `change_id` being integrated
  - `review_comment.target` → a `change_id`
  - `answer.question_id` → the `question`
  - `todo.metadata.blocked_by[]` → prerequisite `task_id`s (dependency edges)
  - `code_change.metadata.base_commit`/`branch`; `feature_complete.changes[]` → change_ids
  - These are framed for "observability and debugging, not for producing merged output."

## What the garden LACKS (absent, verified negative search)  [prior-art]
- No **causality** as a first-class concept (no happens-before, caused-by, vector clocks).
- No **lineage/genealogy/provenance DAG** — the term/concept does not appear.
- No **branching/forking of agent trajectories** (swarm mode = concurrency, not
  trajectory forks; see topic 04 — but NB Gardener has SeedForked/SeedRewound).
- No **conversation/interaction trees** — MUC rooms are flat linear history; the
  project/→feature/→review/ namespace is organizational, not a parent-child event tree.
- No generic parent/child derivation pointer beyond the specific correlation IDs above.

## Freeze/thaw vs. fork/rewind
- **Rewind-by-time: yes** (replay events up to timestamp T; frozen context = linear
  cut of the timeline). **Fork-a-tree: no** — no sibling branches from one node, no
  trajectory merge, no parent/child links between frozen contexts.
- Closest analog to "fork from a point" = resurrection-then-new-feature-room: reload
  a frozen cut into a debug room, then spawn NEW implementers / NEW feature room.
  Branches the *work*, not a formal trajectory tree.
- Open question even flags it: "Inter-feature dependencies — frozen features that
  depend on other features?" is unresolved.

## Implication for Greenwood
- **The flat log is the substrate; the DAG is a projection you must add.** The
  architecture explicitly anticipates derived projections ("any understanding can be
  reconstructed from the stream"). A **provenance/lineage projection** that stitches
  correlation IDs (+ new explicit `derived_from`/`supports`/`contradicts` edges) into
  a genealogy DAG is the architecturally-consistent home for the tree-structure.
- Design decision: store provenance edges *natively in the event envelope* (richer,
  cheaper to project) vs. *reconstruct them in the projection* (keeps the log dumb).
  The paper wants atomic-claim-level edges → likely needs native envelope support
  (see topic 03).
- This is genuinely NEW work vs. the garden — not a revival. Good: it's the
  differentiator, so front-load it (lesson 3 in topic 00).

## Open questions to resolve
- Edge types: just `supports`/`contradicts` (paper) or also `derived_from`,
  `blocks`, `revises`, `forks`?
- Granularity of nodes: message vs. atomic claim (topic 03).
- How does the DAG relate to immutability? (pruning ≠ deletion — see topic 02.)
