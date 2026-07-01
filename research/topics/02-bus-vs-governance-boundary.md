---
doc_type: topic-note
topic: Where governance (screen/block/rollback) belongs — bus vs. projection vs. coordinator
project: Greenwood
tags: [transport-not-merge, projection, governance, rollback, immutability, merge-chibi, boundary]
sources: [spark-to-fire]
created: 2026-06-30
updated: 2026-06-30
status: active
summary: Garden boundaries say governance does NOT belong inside the bus (transport stays dumb). Decisions→Navigator/coordinator role; views→read-only projection; UNDO→compensating events + state re-derivation (NEVER deletion, because the log is immutable). The paper's inline-blocking layer must be reconciled with immutability via "promote-to-confirmed" gating, not erasure.
---

# Bus vs. governance boundary

## The question
The paper's defense is **inline middleware on the message path** that screens,
blocks, and rolls back. Does that belong INSIDE the Greenwood (transport), or as a
layer/projection on top? Where do coordination vs. policy vs. undo live?

## Garden's documented boundaries  [prior-art]
- **"Chattermax is transport, NOT a merge engine."** What the bus DOES: transport
  messages, log to immutable stream, provide queryable history. What it does NOT do:
  merge code, resolve conflicts, produce reconciled output. Event log is "for
  observability and debugging, not for producing merged output."
- A screen/block/rollback layer is essentially a reconciliation/policy engine →
  putting it in the bus violates the boundary. **Keep transport dumb.**
- **Coordination/policy lives in a Navigator role** — the canonical "Merge Chibi:
  coordinate WHEN branches converge, not HOW (git does the merge)" pattern. By
  analogy, governance *decisions* (screen/block) belong in a coordinator role
  beside/above the bus, not in the bus.
- **Projections = read-only role-specific derived views** over the event stream
  (Chalet is the first projection; "local stores are projection caches"). A
  projection can host the *observe/screen-analysis* half of governance (a compliance
  view, an intervention dashboard, an audit view) but by definition does NOT mutate
  the bus.
- **Human oversight** is the documented active-intervention surface: watch all rooms,
  send any message, override agent decisions, pause/resume agents, adjust scope.
- Existing bus-adjacent controls are NOT semantic governance: exec-hook filters match
  *what triggers spawns* (not delivery gating); message validation = schema/malformed
  rejection + rate-limit per JID.

## The immutability constraint (sharpest)  [prior-art]
- "State is DERIVED, not STORED. The immutable event log IS the source of truth."
- ∴ you **cannot "roll back" by deleting/un-sending** a message. The
  architecturally-consistent rollback = **emit a compensating/corrective event and
  re-derive state by pointing to an earlier event range.** Rollback is a
  state-reconstruction operation, not a transport mutation.

## Reconciling with the paper
The paper's layer "blocks" a message before it enters downstream *trusted context* —
that is compatible with immutability if reframed:
- The bus **still logs everything immutably** (including rejected/quarantined claims).
- "Pruning" = **never promoting a claim to `confirmed`/trusted** + emitting a
  compensating event + re-deriving downstream state. The faulty branch stays in the
  log (audit) but is excluded from the confirmed lineage. This matches the paper's
  confirmed-vs-unverified node distinction exactly.
- The paper keeps topology `A` unchanged and "intervenes only on the message path" —
  i.e. it is itself a coordinator-style middleware, not a rewiring. Consistent with
  "transport stays dumb; coordinator decides."

## Proposed placement for Greenwood (synthesis — to validate)
- **Bus (transport + immutable log):** carries events, never decides. Logs ALL claims
  including rejected ones.
- **Lineage projection:** maintains the genealogy DAG + confirmed/unverified state +
  risk/centrality scoring (read model). Hosts the observe/screen-analysis half.
- **Governance coordinator (Navigator-style role):** consumes the lineage projection,
  makes promote/block/verify/rollback DECISIONS, and enacts them by emitting
  compensating/quarantine events back onto the bus. This is the "Merge Chibi:
  coordinate not merge" analog → "Governor: decide not delete."
- **Human oversight** retains override/pause/scope authority above all of it.

## Open questions
- Is the governance coordinator in-band (an agent on the bus) or a privileged
  out-of-band service? Paper runs it inline (latency cost); garden pattern → a role.
- Latency: inline synchronous screening adds per-hop latency (paper measures this).
  Can Greenwood do async screening with a "quarantine until confirmed" gate instead?
- Does "promote to confirmed" gating require consumers to honor a trust flag (trust
  the projection), and what stops an agent from reading unconfirmed claims directly?
