---
doc_type: design-sketch
topic: Agent resume from Kafka after pod reschedule, preserving LLM prompt-cache hits
project: Greenwood
tags: [resume, kafka, event-sourcing, prompt-caching, k8s, snapshots, determinism, replay, anthropic]
sources: [claude-api-skill, engineering]
created: 2026-06-30
updated: 2026-06-30
status: draft
summary: An agent's trajectory is Kafka events keyed by agent-session (ordered). Resume = replay-from-snapshot to rebuild the messages array DETERMINISTICALLY, so the reconstructed prompt prefix is byte-identical and hits the content-addressed LLM cache from a different pod. Anthropic cache is content-addressed (not session-bound) → any host reproducing identical bytes within TTL hits it. Use 1h TTL to survive reschedules.
---

# Agent resume + LLM cache continuity

## The scenario
Agent runs in a k8s pod; pod gets rescheduled (eviction, OOM, node drain, crash).
A new pod must resume the agent from Kafka with full context — and do so without
losing the LLM provider's prompt cache (re-prefilling a long context is slow + costly).

## Two stacked problems
1. **State reconstruction** — rebuild the agent's exact conversation/tool state from
   the event log.
2. **Cache continuity** — make the reconstructed prompt hit the provider's warm cache.

## Grounding: how Anthropic prompt caching actually works  [claude-api skill]
- **Content-addressed prefix cache, NOT session-based.** The Messages API is
  *stateless* — there is no server-side session to "resume into." The cache key is
  derived from the **exact rendered bytes** of the prompt (render order:
  `tools → system → messages`) up to each `cache_control` breakpoint, plus the model.
- **Therefore a DIFFERENT host/pod hits the same warm cache iff it reconstructs the
  identical byte prefix**, on the same model + org/workspace, within TTL. The cache
  is not tied to a machine, process, or session id. This is the property that makes
  Kafka-based resume work with caching at all.
- **TTL:** 5 min default (`ephemeral`, 1.25× write cost); **1 hour** via
  `cache_control:{type:"ephemeral", ttl:"1h"}` (2× write). Refreshed on each hit.
- **Any byte change anywhere in the prefix invalidates everything after it.** Silent
  invalidators: `datetime.now()`, UUIDs, unsorted JSON, a varying tool set.
- Min cacheable prefix: 4096 tokens (Opus 4.8) / 2048 (Sonnet 4.6). Verify with
  `usage.cache_read_input_tokens`.
- **Model is `claude-opus-4-8`** (1M context) for this project's agents.

### Consequence for "send it to the same session"
For Anthropic there is no session handle to carry — **"resume the session" = reproduce
the identical prompt prefix.** (Contrast: server-stateful surfaces — Anthropic
**Managed Agents** sessions, or OpenAI Responses `previous_response_id` — DO carry a
server-side session with server-managed caching/compaction; there you'd persist the
session id and resume its event stream instead of rebuilding bytes. Managed Agents is
a real alternative architecture worth weighing vs. self-hosting the loop on Kafka.)

## Design (self-hosted loop on Kafka)

### State model
- The agent's whole trajectory = events on Kafka, **keyed by `agent_session_id`**
  (or `lineage_root`) so all its events land in **one partition → total order**.
  (NB: Chattermax keyed by random event-UUID and *explicitly deferred* room/agent
  keying, so it had **no per-agent ordering** — do NOT copy that; key for ordering.)
- Periodic **snapshot event** = the fully-serialized messages array (inline, or a
  pointer to object store) + the **Kafka offset** it reflects + the cache-breakpoint
  positions. Bounds replay cost (the garden flagged "snapshot strategy" as an open
  event-sourcing concern; Kafka + snapshots solves it).

### Resume sequence
1. **New pod acquires the agent.** Two patterns:
   - *Consumer-group-as-failover:* partition-per-agent → Kafka rebalance reassigns the
     partition to a live pod automatically on death. Elegant, but partitions are
     finite (~thousands) so it caps concurrent agents.
   - *Claim/work-queue:* a `control`-topic claim ("agent X needs a worker"); pods
     claim sessions. Scales past partition limits. (Likely the general answer;
     partition-per-agent for small fleets.)
2. **Rebuild state:** load latest snapshot, replay events after its offset, reconstruct
   the in-memory messages array.
3. **Re-issue next LLM call** with the *same* `cache_control` breakpoints → cache hit,
   if within TTL.

### The hard requirement: deterministic serialization
Cache continuity hinges on the reconstructed prompt being a **pure function of the
event log** — byte-identical regardless of which pod builds it:
- Freeze the system prompt; **no** timestamps / UUIDs / pod-ids / nonces in the cached
  prefix. Put anything volatile *after* the last breakpoint.
- **Sort tool definitions deterministically**; never vary the tool set by pod.
- Canonical JSON (sorted keys) for any serialized structures in the prefix.
- Place `cache_control` breakpoints at deterministic positions (after system+tools;
  after the last stable turn).
- If any of these drift between the old and new pod, `cache_read_input_tokens` = 0 and
  you silently pay full prefill. This is the #1 failure mode — test it.

### TTL choice
Use **`ttl:"1h"`**. Pod eviction → reschedule → image pull → warmup routinely exceeds
5 min; the 5-min cache would be cold by the time the new pod issues its first call.
1h survives typical reschedules. Past TTL: cache cold, correctness preserved, pay one
prefill. (Refresh-on-use keeps it warm while the agent is actively looping.)

### In-flight call at crash
If the pod dies mid-LLM-call, the response was never persisted to Kafka. On replay the
new pod re-issues the last request: the prefix hits cache (cheap prefill), generation
is redone. To recover *partial* generations, **stream output deltas into Kafka** (seq
numbers, à la Chattermax's `chibi:stream` `stream_id`/`sequence`/`complete`) so a
resume can salvage the partial. Simpler: accept re-generation of the last turn.

### Idempotency / no double-emit on resume
Re-emitting events during replay must not duplicate. Track **last-committed offset per
agent** (Kafka consumer offset commit), or use **deterministic content-hash event ids**
so re-emission is idempotent. (Chattermax used the event-UUID-as-key for *dedup*; a
content-hash id generalizes that to replay-safety.)

## Open questions
- Partition-per-agent vs. claim/work-queue — pick by expected concurrent-agent count.
- Snapshot cadence (every N turns? every M tokens?) vs. replay cost tradeoff.
- Does the cached prefix include tool *results* (large)? If so, snapshotting the
  serialized array (not just replaying) keeps the bytes stable and cheap to rebuild.
- Self-host-on-Kafka vs. adopt Anthropic **Managed Agents** (server-side sessions,
  server-managed cache/compaction, hosted container) — the latter removes the
  reconstruct-identical-bytes burden but gives up Kafka-native control of the spine.
  Decision doc needed (relates to topics 02/05).
- Multi-provider: an OpenAI-Responses-style backend would persist `previous_response_id`
  in a snapshot event instead of rebuilding bytes — envelope should carry an optional
  `provider_session` field to support both models.
