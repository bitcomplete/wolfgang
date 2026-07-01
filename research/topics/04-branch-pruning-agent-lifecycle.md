---
doc_type: topic-note
topic: Pruning an agent branch (not just a claim) — lifecycle, swarm, fork/rewind
project: Greenwood
tags: [agent-lifecycle, swarm, freeze-thaw, resurrection, fork, rewind, failure-recovery, gardener-seed, pruning]
sources: [spark-to-fire]
created: 2026-06-30
updated: 2026-06-30
status: active
summary: The garden can fan-out (swarm), freeze a HEALTHY branch, and resurrect it for inspection — but CANNOT detect/kill/rewind/fork/rollback a FAULTY branch (failure recovery is an explicit open question; abandon/rollback of a feature room is undocumented). KEY: Gardener's agent-as-seed model DOES have SeedForked/SeedRewound/SeedPromoted — the missing prune primitive lives there, not in Chattermax.
---

# Branch pruning & agent lifecycle

## Two senses of "branch"
1. **Claim branch** — a subtree of the lineage DAG descending from a faulty atomic
   claim (paper's domain; see topics 01/03). Prune = quarantine claim + dependents.
2. **Agent branch** — a spawned agent / swarm worker / feature-room trajectory that
   went wrong. Prune = detect → stop/kill → discard or rewind → reclaim. THIS note.

## Garden state on the agent-branch sense  [prior-art]
- **Swarm mode = fan-out, NOT divergent branches.** One Chibi process, multiple XMPP
  resources (`implementer-1@chattermax/worker-1..N`), each a separate room
  participant doing parallel work. Mechanics deferred to Chibi owner (Open Q #4).
  No notion of these workers diverging in state and being independently kept/discarded.
- **Freeze/thaw/resurrection = forward replay of a HEALTHY branch for inspection.**
  Freeze is triggered by `feature_complete` and freezes participants who posted
  `code_change` — it's archival of *completed* work, not discard of *bad* work.
  Resurrection respawns frozen Chibis into a `debug/` room (read-only; `/unfreeze`
  for write) where they can be questioned, and you may spawn NEW implementers / a NEW
  feature room from there.
- **Rewind/fork of a live agent = enabled-in-principle, undesigned-in-practice.**
  Event sourcing could replay to event N and branch, but no spec describes rewinding
  a live agent or forking it at a past state. Current freeze is "best-effort
  snapshots, not precise replay."
- **Detect-it-went-wrong → UNDESIGNED.** "Failure recovery — what happens when
  Implementer Chibi crashes mid-work?" is a verbatim OPEN QUESTION. No crash
  detection, kill/stop primitive, or recovery procedure. Containment levers
  (rate-limiting #5, cost-management #6) are also open questions.
- **Abandon/rollback a feature room → NOT DOCUMENTED AT ALL.** Architecture is
  completion-oriented (work_available → … → feature_complete → freeze). No path to
  abandon mid-work, discard accumulated changes, or kill a trajectory without freezing.

## The key cross-link: Gardener already has the prune primitives  [prior-art]
Gardener's **agent-as-seed** model has domain events **`SeedForked`**,
**`SeedRewound`**, **`SeedPromoted`** (plus `SleepCycleStarted`, `OPLoRAComputed`).
So the fork/rewind/promote lifecycle that Chattermax lacks **exists in the garden —
in Gardener, not the bus.** Gardener is also the only actively-shipping garden-core
component and is framed as the cultivation/lifecycle layer. This strongly suggests
the agent-branch lifecycle (fork/rewind/prune) is a *cultivation-layer* concern, while
the claim-branch lineage is a *bus/governance* concern — two coupled tree structures.

## What "pruning an agent branch" requires (composition)
- The branch exists: swarm resource OR feature-room trajectory — *documented*.
- Detect it went wrong — *undesigned* (Open Q #7).
- Stop/kill it — *undesigned* (no kill primitive; rate-limit/cost are open).
- Discard work without archiving — *not documented* (only success→freeze exists).
- Rewind/fork to a known-good state — *enabled by event sourcing, undesigned;* but
  Gardener's `SeedRewound`/`SeedForked` are the existing model to borrow.

## Implications for Greenwood
- **Two coupled trees:** (a) lineage DAG of claims (govern/prune claims), (b) agent
  trajectory tree (fork/rewind/prune agents). The paper covers (a); Gardener's seed
  events cover (b). Greenwood likely needs BOTH, with a mapping: a confirmed-false
  claim subtree → triggers rewind/abandon of the agent branch that produced/relied on
  it.
- **Design the faulty-branch lifecycle explicitly** — add the states the garden left
  open: `abandoned`, `rolled_back`, `quarantined`, plus crash/timeout detection and a
  kill/stop primitive. This is exactly the garden's biggest gap and a core Greenwood
  differentiator.
- **True rewind/fork depends on precise event-replay** (not best-effort snapshots).
  If Greenwood commits to event sourcing, wire replayable state from day one so
  rewind/fork is real, not aspirational (garden lesson: don't ship the snapshot bridge).
- **Borrow Gardener's seed vocabulary** (Forked/Rewound/Promoted) for the agent-branch
  tree rather than inventing — it's the garden's most evolved lifecycle thinking.

## Open questions
- Granularity of an "agent branch": a swarm worker? a feature room? a single agent
  turn? The unit of pruning must be defined.
- When a claim is rejected, what's the blast radius on agent branches — rewind only
  the relying agent, or all downstream adopters (paper's adoption set)?
- Reconcile rewind with immutability (topic 02): rewind = compensating event +
  re-derive, fork = new branch pointer, never deletion.
