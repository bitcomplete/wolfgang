---
doc_type: design-sketch
topic: Graft — harness adapters; making Greenwood harness-agnostic
project: Greenwood
tags: [graft, adapters, harness, protocol, conformance, sdk, claude-code, codex, pi.dev, hermes, determinism]
sources: [decisions, topics-06-08, engineering]
created: 2026-07-01
updated: 2026-07-01
status: draft
summary: Greenwood's only contract with the outside world is its event envelope + a lifecycle protocol. Any agent harness (Claude Code, Codex, pi.dev, Hermes, …) plugs in through a Graft — an adapter that translates the harness's native loop onto that protocol. Ships as a protocol spec + per-language SDKs + a conformance suite + reference grafts. Conformance is two-tier: correctness (resume rebuilds correct state — required) and cache-continuity (byte-identical prompt — best-effort).
---

# Graft — harness adapters

Greenwood should not be tied to one agent harness. Its only contract with the outside
world is the **event envelope** (topic 08 §1) plus a **lifecycle protocol**. A harness
plugs in through a **Graft** — an adapter that translates the harness's native loop onto
that contract. The bus never knows which harness produced an event. (Graft = joining a
scion onto a rootstock: harness = scion, Greenwood = rootstock, adapter = graft.)

## The contract a Graft implements
1. **Translate actions → events.** Map the harness's thoughts / tool calls / tool results
   / outputs to Greenwood's typed events. Surface claims + provenance either from the
   harness (if claim-aware) or via Greenwood's decomposition over raw output — so a
   harness does **not** have to be claim-aware to be adaptable.
2. **Lifecycle.** `init(context) → session`, `step(session) → events`,
   `apply_tool_result(session, result)`, `snapshot(session) → ref`,
   `resume(snapshot, events) → session`, `stop(session)`. This is what lets Greenwood
   spawn, pause, snapshot, and resume the harness on another host.
3. **Accept a handoff.** Turn a Greenwood handoff payload (Tier-0 synthesis + expandable
   slice, topic 07 §5 / D2) into the harness's starting context.

## Conformance is two-tier (be honest about this)
- **Correctness (required):** `resume(snapshot + replayed events)` rebuilds the harness's
  *correct* state. Every graft must pass this.
- **Cache-continuity (best-effort):** it rebuilds the prompt **byte-identically**, so the
  content-addressed prompt cache (topic 06) hits on a fresh host. Harnesses whose prompt
  serialization we don't fully control may miss this — they still resume correctly, they
  just pay a prefill. Grafts advertise which tier they meet.

## Deployment
- **Sidecar (default):** the graft is a process speaking the Graft protocol
  (gRPC over the event envelope + a lifecycle service). Fits harnesses that are their own
  binaries/CLIs in their own languages (Claude Code, Codex).
- **In-process library:** for a harness embedded in the runtime's language. Same wire
  contract.
- **Tool execution:** either the harness runs its own tools and reports
  `tool_call` + `tool_result` events, or Greenwood surfaces the `tool_call` and the host
  runs it and feeds the result back. The recorded events are identical either way — which
  is what hermetic eval replay (Coldframe) depends on.

## What ships
- **Graft protocol spec** — the Protobuf event envelope + the gRPC lifecycle service.
  Language-neutral, versioned (Schema Registry).
- **Graft SDKs** (Rust, TypeScript, Python, Go) — a base that handles the Kafka wiring
  (produce/consume, offset management, snapshot storage) so an author writes only the
  harness-specific translation + lifecycle.
- **Conformance suite** — round-trip event mapping, correct-state resume, byte-identical
  resume, handoff injection. Passing it means the graft works with all of Greenwood
  (resume, handoff, governance, evals) with no per-harness special-casing.
- **Reference grafts** — Claude Code, Codex, pi.dev, Hermes Agent, as worked examples.

## Open questions
- **Snapshot ownership.** Does the graft serialize harness state (opaque blob) or does
  Greenwood reconstruct purely from events? Likely a mix: events are the source of truth;
  the snapshot is an optimization the graft may provide.
- **Determinism knobs.** What does the SDK give an author to hit the byte-identical tier
  (canonical serialization helpers, breakpoint placement, "no wall-clock in prefix" lint)?
- **Capability negotiation.** A graft advertises capabilities (native claims? streaming?
  self-executes tools? cache-continuity tier?) so Greenwood adapts behavior per harness.
- **Versioning.** Harness upgrades change serialization → resume across a harness-version
  bump may drop to correctness-only. Track harness version on events.
