---
doc_type: design-sketch
topic: Agent eval — feedback-seeded, replay-based evals built on the event bus
project: Greenwood
tags: [eval, feedback, replay, frozen-context, hermetic, harness, regression, llm-judge, chizu, hotspots, event-sourcing]
sources: [decisions, topics-06-08, spark-to-fire]
created: 2026-06-30
updated: 2026-06-30
status: draft
summary: Because everything is event-sourced (P0) and replay is deterministic (topic 06), a failure is ALREADY a replayable frozen context. Human feedback ("thumbs down; should've done X not Y") is registered as an event; an eval-builder freezes the exact context at that point + derives an assertion; the eval loop = replay-through-harness → must-fail (reproduce) → tweak harness → test → iterate until pass → add to regression suite. Reproduction is HERMETIC (recorded tool_results as fixtures; only the model varies) → evals are statistical (pass-rate ≥ threshold). Builds directly on P3 (resume/replay) + the lineage DAG. This is the garden's Hotspots/Chizu refinement loop, on the bus.
---

# Agent eval — feedback → replay → eval → refine

## Thesis
The event-sourced spine gives evals almost for free: an agent's exact context at any
moment is a **frozen context = pointer to an event range**, deterministically
reconstructable by replay (the *same* machinery as resume, topic 06 / P3). So an eval is
just: **replay the frozen context up to the flagged action, run the (possibly tweaked)
harness, assert the corrected behavior.** Human feedback seeds it; the loop refines the
harness until the assertion passes; the passing eval joins a regression suite. (This is
the garden's "soil testing lab" (Hotspots/ASH) + Chizu's automated
validate→prune→improve loop — realized on the bus, seeded by human feedback.)

## 1. Feedback is an event (registered in the agent log stream)
A `Feedback` event on the bus (topic `agent-bus.feedback`, keyed by `lineage_root` so it
co-locates/orders with the trajectory it critiques; provenance `derived_from` the target
events). Sources: humans (thumbs up/down + correction), implicit signals (task/tests
failed, PR rejected, user redid the work), and the **verifier's own `REJECTED`
transitions** (already-labeled failures). All become eval seeds.
```proto
message Feedback {
  repeated string target_ref = 1;   // event_id(s)/claim_id/turn the feedback is about
  string agent_session_id = 2; string lineage_root = 3;
  enum Signal { THUMBS_UP=0; THUMBS_DOWN=1; RATING=2; STRUCTURED=3; } Signal signal = 4;
  string observed = 5;              // "did Y" (or derivable from target)
  string expected = 6;             // "should have done X" (NL, optionally structured)
  string author = 7;               // human id / automated source
}
```

## 2. Eval-case synthesis (an eval-builder process, beside the bus)
Consumes `Feedback`; for each, produces an **EvalCase** (itself an event — P0; evals are
event-sourced + versioned):
- **Freeze the context:** the event range from `Snapshot`→`target_ref` offset =
  the exact input the agent saw at the flagged action.
- **Derive the assertion** from `expected`: structured (the corrected tool call/claim/
  decision), or a **rubric for an LLM-judge** ("does the output do X and not Y?"), or a
  checkable predicate. (Mirrors the paper's rubric + LLM-judge pattern.)
- **EvalCase** = `{frozen_context_ref (range), input, assertion|rubric, provenance→
  feedback→trajectory}`.

## 3. The loop (reproduce → tweak → test → iterate)
1. **Reproduce (must fail first):** replay the frozen context through the *current*
   harness → check assertion. It should **fail** — confirming the eval actually captures
   the bug. (An eval that passes on first run against the buggy harness is mis-specified.)
2. **Tweak the harness:** change a versioned harness bundle (system prompt / tool set /
   verification policy / model / effort / skills). By human, or by a meta-agent
   ("harness engineer") that itself runs on the bus — its tweaks are events (P0).
3. **Test:** re-run the eval → assert.
4. **Iterate until pass**, then **add to the regression suite**: every future harness
   change re-runs the suite → CI for agent behavior; you can't silently regress a fixed
   case.

The eval-runner is just an agent spawned in **replay/eval mode pointed at a frozen
context** — i.e., the resume machinery (T3.2) + an assert-against-rubric wrapper. (This
is Chattermax's `--mode debug` resurrection, repurposed for evals.)

## 4. Hermetic replay (the key reproduction decision)
Replaying "the exact context" reproduces the exact **input**, but tools that hit the
outside world won't reproduce. Default to **hermetic replay**: feed the **recorded
`ToolResult` events** (they're already in the log as evidence — topic 03) back as
fixtures instead of re-executing tools. Deterministic inputs; only the model varies.
(Non-hermetic "live tools" mode is an option for integration evals, but flaky.)

## 5. Evals are statistical (LLMs are stochastic)
Even with a byte-identical input, the model's output varies run-to-run. So an eval isn't
"produces exactly X" — it's "**satisfies the assertion/rubric**," run **N times with a
pass-rate ≥ threshold** to handle flakiness. (This is the paper's `S(t)`/coverage
framing turned into a pass-rate metric per harness version.)

## 6. Genealogy leverage (why building on THIS bus matters)
- Feedback on a claim → query `descendants(claim_id)` (lineage DAG) → the **blast
  radius**: every downstream claim/agent that relied on the flagged one. Target the eval
  at the **root cause**, not the symptom.
- **Cluster similar failures** across trajectories (shared tags/subgraph shape) → one
  eval can cover a class of bugs, and a harness fix can be validated against the cluster.
- Full traceability: EvalCase `derived_from` Feedback `derived_from` the exact agent
  moment — you can always walk an eval back to the human complaint and the frozen context.

## 7. Harness versioning
A **harness version** = a versioned bundle (system prompt, tools, verification policy,
model, effort, skills). Evals run *against a harness version*; suite pass-rate before/
after a change is how you know a tweak helped. Harness versions and eval runs are events
→ the whole refine loop is auditable and replayable.

## 8. Open questions
- **Assertion representation** — how much structured vs. rubric/LLM-judge? Structured is
  crisp but brittle; rubric is flexible but needs a judge model (and a judge can be
  wrong — verify the judge? P0 says its verdicts are events too).
- **Flakiness threshold / N** — per-eval or global? How to distinguish a real regression
  from noise.
- **Meta-agent autonomy** — human-in-loop tweaking vs. an autonomous "harness engineer"
  that proposes tweaks and opens them for review (its tweaks are events, so governable).
- **Eval selection at scale** — which subset of the suite runs on each harness change
  (full suite is expensive); relevance by tag/subgraph like handoff selection (D2).
- Relationship to `evidence_ref` drift — if a hermetic fixture (recorded tool result)
  becomes stale vs. the real world, the eval tests past-reality; acceptable for behavior
  regression, note it.
