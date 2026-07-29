---
title: "Research round 2 — per-agent identity and signed agent actions"
date: 2026-07-29
topic: agent-identity-signed-events
status: draft
project: Greenwood
tags: [r2, identity, signing, envelope, T1.1, buzz]
---

# Per-agent identity and signed agent actions (research round 2, 2026-07-29)

Prompted by the Buzz synthesis (competitor-hive.md §4/§7): Buzz gives each agent its own
Nostr keypair and signs every event (NIP-01), yielding a per-agent audit trail. Question:
should Greenwood's T1.1 event envelope carry an agent signature — and what identity
fields must the greenfield schema reserve on day one so identity can be added without
migration?

Existing envelope (topics/08 §1): `event_id` (canonical content hash; ts/producer/epoch
excluded), `schema_version`, `ts`, `producer` (plain string), `agent_session_id`,
`lineage_root`, `session_epoch`, payload oneof. Related prior findings: DC-3
originating-human authority field (gates deep dive F6, OWASP ASI03); key-per-agent
Anthropic API keys for spend attribution (cost deep dive); OTel GenAI semconv names
(DC-4).

## Scored findings

| # | Finding | Relevance to Aug-16 R1 |
|---|---------|------------------------|
| F1 | **The envelope's `event_id` canonical-hash spec is already the signing pre-image**: any future signature signs `event_id`; reserving one `Signature{key_id, scheme, sig}` field + writing that rule makes signatures a later key-management project, never a schema migration | **HIGH** |
| F2 | **Identity discipline ≠ cryptography**: the audit value of Buzz's model at R1 scale is captured by a structured `Principal{id, kind, runtime}` producer field (aligned with `gen_ai.agent.id`, DC-4) + per-agent transport credentials; per-event keypair signatures add nothing a trusted-runtime deployment can verify | **HIGH** |
| F3 | **Impersonation control belongs at the broker, not the envelope**: per-agent Kafka principals (SASL/mTLS) + ACLs binding principal → producer id close the forged-`producer` hole without envelope crypto; schema just needs the principal id to bind against | **HIGH** |
| F4 | **Originating-human field needs integrity protection to mean anything** (OWASP ASI03 / AWS Cedar HMAC-context pattern): reserve `mac` inside `originating_human` now; a lying intermediate agent is the R1-realistic attacker | **HIGH** |
| F5 | **Sign decisions, not thoughts**: if anything gets signed early, it's `ApprovalGranted` / `PolicyDecision` / originating-human context — highest audit value per unit of key-management surface | **HIGH** |
| F6 | Nostr NIP-01 (Buzz) = canonical-serialization hash + sig over hash + pubkey-as-identity; architecture transfers, threat model does not (untrusted relay vs. trusted bus) | MEDIUM |
| F7 | Session/lineage-root granularity signing (Merkle root of a segment's event_ids attested at session close; in-toto/DSSE or Sigstore shape) = tamper-evidence for exported audit ranges at ~1 signature/session | MEDIUM |
| F8 | SPIFFE/SPIRE per-agent SVIDs; Microsoft agent-governance control plane ships SPIFFE-compatible agent identities (Ed25519 + ML-DSA-65) — the industry pattern for *multi-service* agent fleets | MEDIUM |
| F9 | OAuth for agents: client-credentials per agent + RFC 8693 token exchange for delegation chains; MCP authorization = OAuth 2.1 resource servers — tool-boundary machinery, not bus-envelope machinery | MEDIUM |
| F10 | NHI governance (OWASP NHI Top 10, CSA agentic-identity work 2025): inventory, one identity per workload, short-lived creds, human owner per NHI — checklist for M2 hardening | MEDIUM |
| F11 | Per-agent cloud identities (IAM role / k8s ServiceAccount per agent) so platform audit logs decompose per agent — same axis as key-per-agent Anthropic keys | MEDIUM |
| F12 | Kafka has no native message-level signing; e2e payload signatures are app-layer (header over canonical bytes); CPU cost negligible at Greenwood scale — cost is key mgmt + canonicalization (which T1.1 already pays) | MEDIUM |
| F13 | C2PA — media provenance only; not envelope-relevant | LOW |
| F14 | Quantum-safe agent identity (ML-DSA), web-of-trust agent reputation | LOW |
| F15 | Sigstore keyless per-event signing (Fulcio cert + Rekor entry per event) — wrong-shaped at event granularity | LOW |

## 1. Workload identity applied to agents

- **SPIFFE/SPIRE** issues short-lived workload identities (SPIFFE ID URI + X.509/JWT
  SVIDs) via node+workload attestation — no long-lived secrets. The natural mapping:
  one SPIFFE ID per agent (`spiffe://greenwood/agent/<name>`), SVID used for mTLS to
  Kafka and for JWT-SVID claims in the envelope. Published 2025-26 AI-agent patterns
  exist and treat SPIFFE as the presumed agent-identity substrate:
  https://www.hashicorp.com/en/blog/spiffe-securing-the-identity-of-agentic-ai-and-non-human-actors ;
  https://www.paloaltonetworks.com/blog/identity-security/ai-agent-security-spiffe-machine-identity/
  [reported — carried from pass-1 gates research]. The Microsoft agent-governance
  control plane likewise ships SPIFFE-compatible agent identities (Ed25519 + ML-DSA-65)
  with hash-chained audit:
  https://developer.microsoft.com/blog/securing-mcp-a-control-plane-for-agent-tool-execution/
  [verified in gates deep dive].
- **The flagged gap:** SPIFFE proves *who* an agent is, not *what it may do* —
  authorization stays a separate layer (Cedar/policy work, DC-2):
  https://nhimg.org/articles/spiffe-and-ai-agent-identity-expose-the-next-authorization-gap/
  [reported]. Identity work does not substitute for the gates workstream.
- **NIST NCCoE concept paper (reported as Feb 2026)** names SPIFFE/SPIRE + OAuth 2.0 +
  zero trust as candidate AI-agent identity standards (via
  https://www.digitalapplied.com/blog/agent-identity-credentials-non-human-access-2026-playbook —
  not independently verified against nist.gov).
- **Cloud-native equivalent:** one IAM role / GCP service account per agent workload
  (workload identity federation), so cloud audit logs decompose per agent — same move
  as key-per-agent Anthropic keys (cost deep dive §3.3). On bc-prod (k8s), the free
  version is one ServiceAccount per agent + per-principal Kafka credentials.
- **Lightest-weight version for a tiny company:** identity *claim* in the envelope
  (`producer` as a stable, registered agent ID) + per-agent transport credentials
  (Kafka SASL/mTLS principal per agent, broker ACLs binding principal → allowed
  topics). The runtime asserts identity; the broker authenticates it. A
  keypair-per-agent with per-event signatures only pays off when a verifier *outside*
  the trusted runtime needs to check authorship.

## 2. Signed events / provenance

- **Kafka has no native message signing** — integrity/authn is transport-level (mTLS,
  SASL) plus broker ACLs; end-to-end payload signatures are an application-layer
  pattern (signature in a header over the canonical payload bytes). At Greenwood scale
  (single-digit agents, modest event rates) Ed25519 sign+verify overhead is trivial
  (~tens of microseconds each); the real cost is **key management + canonicalization
  discipline**, not CPU.
- **Canonicalization is the hard part Greenwood already solved:** T1.1's canonical-hash
  spec (pinned field order, ts/producer/epoch excluded, cross-language test vectors)
  is exactly what a signature needs as its message. Sign `event_id` (the content hash)
  rather than raw bytes → signature scheme inherits the hash spec for free.
- **Nostr NIP-01 as prior art**
  (https://github.com/nostr-protocol/nips/blob/master/01.md [stable spec, training
  knowledge]): event = `{id = sha256(canonical JSON serialization), pubkey, sig =
  schnorr(id)}` — identical architecture to what Greenwood would build: canonical
  serialization first, hash, signature over the hash, identity = pubkey. Buzz inherits
  this wholesale (agent key via `BUZZ_PRIVATE_KEY`; competitor-hive.md §4). The part
  that does NOT transfer: Nostr signs because relays are untrusted and anyone can
  connect; Greenwood's broker is operated by the same party as the runtime, and even
  Buzz's own reviewers concede the result is "tamper-evident, not tamper-resistant."
- **Sigstore** (https://www.sigstore.dev/ [stable]) = keyless signing (Fulcio
  short-lived certs bound to OIDC identity + Rekor transparency log). Built for
  artifacts; per-event use is wrong-shaped (a Rekor entry per bus event is absurd at
  event rates), but **session/lineage-root granularity** fits: sign a manifest of a
  lineage segment (Merkle root of event_ids) once per session close — one signature,
  tamper-evidence for the whole range, and keyless means no per-agent key custody.
- **in-toto attestations** (https://github.com/in-toto/attestation [stable]; Statement
  + predicate in a DSSE envelope) — the right shape if Greenwood ever exports "what
  did agent X do" as compliance evidence: one attestation over a range of event_ids,
  not per-event. Pairs with the observation (gates deep dive) that a hash-chained
  audit file is redundant with Kafka's log until events are *exported*.
- **Sigstore where it already fits (local precedent):** trellis Sigstore-signs agent
  *instruction files* (prompts, CLAUDE.md, registry.yaml) in CI and verifies at runtime
  (repo-trellis.md L6 attestation) — signing agent **configuration** in CI is the
  correctly-shaped Sigstore use for the garden, orthogonal to event signing.
- **C2PA** — content provenance for media; relevant only if garden agents publish
  artifacts where downstream consumers check C2PA. Not envelope-relevant.
- **Granularity economics:** per-event signing = per-event key access on the runtime
  host (key effectively as exposed as the runtime). Signing at lineage-root/session
  close or on **high-stakes event types only** (approvals, policy decisions,
  originating-human context) captures most audit value at ~none of the operational
  surface.

## 3. Emerging agent-identity standards 2025-2026

- **OWASP** — the agentic top-10's **ASI03 Identity & Privilege Abuse** is already
  driving DC-3 (verified via
  https://aws.amazon.com/blogs/security/enforce-least-privilege-authorization-in-multi-agent-ai-chains-using-cedar/
  in the gates deep dive): across delegation hops "an agent can potentially act beyond
  what the originating user authorized." OWASP **NHI Top 10** (2025,
  https://owasp.org/www-project-non-human-identities-top-10/ [training knowledge, not
  re-verified today]): secret leakage, overprivileged NHIs, no rotation, human-NHI
  blurring. Concrete guidance level: inventory NHIs, one identity per workload,
  short-lived credentials, every NHI has a human owner. OWASP LLM06 Excessive Agency
  adds: least privilege enforced *externally*, never by the model (pass-1 §4).
- **CSA** — "Agentic Identity Governance Framework v1"
  (https://labs.cloudsecurityalliance.org/agentic/agentic-identity-governance-framework-v1/
  [reported, pass-1]) plus the "NHI governance vacuum" framing; NHI:human ratios
  reported at 80-144:1. This is the most concrete NHI-governance guidance found:
  directory-registered agent identities, ownership, lifecycle (provision → rotate →
  retire), attribution to a responsible human — i.e., exactly the `Principal` +
  `originating_human` pair, no cryptography required.
- **OAuth/OIDC for agents:** client-credentials grant per agent (each agent = OAuth
  client), **RFC 8693 token exchange** for on-behalf-of delegation chains (gates deep
  dive F13; https://datatracker.ietf.org/doc/html/rfc8693). IETF **WIMSE** WG
  (Workload Identity in Multi System Environments,
  https://datatracker.ietf.org/wg/wimse/about/ [training knowledge]) is where
  workload-to-workload identity chaining is being standardized; agent-specific OAuth
  profiles were still draft-stage as of early 2026 [could not re-verify live —
  classifier outage; see Sources note].
- **MCP authorization spec:** the 2025-06-18 revision made MCP servers OAuth 2.1
  resource servers (PKCE, resource indicators RFC 8707). The **MCP 2026-07-28 release**
  (already cited in the gates deep dive: https://blog.modelcontextprotocol.io/posts/2026-07-28/ ;
  https://stacktr.ee/blog/mcp-2026-spec-changes) restructures elicitation
  (`InputRequiredResult` + URL-mode for OAuth consent) and **deliberately leaves token
  vaulting, JIT consent, RBAC, and audit to the runtime layer — i.e., the bus owns
  them**. Relevant to Greenwood at the tool boundary, not the bus envelope.
- **NIST:** nothing NHI-specific verified beyond the NCCoE concept paper (§1); zero
  trust (SP 800-207) and AU-9/AC-3 controls (already cited in gates work) remain the
  applicable baseline.
- Vendor reality check: Microsoft Entra Agent ID, HashiCorp Vault agentic-identity
  validated pattern
  (https://developer.hashicorp.com/validated-patterns/vault/ai-agent-identity-with-hashicorp-vault,
  gates deep dive §5) — the industry direction is "agents get first-class directory
  identities with owners, lifecycle, and short-lived credentials," **not** "agents get
  raw long-lived keypairs." Buzz's keypair-per-agent is the outlier, forced by Nostr's
  transport, not the emerging NHI consensus.

## 4. The threat model, honestly (analysis)

Greenwood's deployment shape vs. Buzz's:

| Property | Buzz | Greenwood R1 |
|---|---|---|
| Trust boundary | Multi-party: any client talks to a relay; the relay is *untrusted* | Single company, single Kafka cluster; runtime + bus operated by the same party |
| Who can write | Anyone with a WebSocket connection — identity **must** be in the message because the transport proves nothing | Only workloads holding cluster credentials; Kafka ACLs/mTLS can already bind producer → topic |
| Verifier | Every client independently verifies NIP-01 signatures | The bus itself (Grieve, folds) — which is also the party operating the runtime |

What per-event agent signatures actually buy in Greenwood's shape:

- **Impersonation by a compromised *component*** — REAL but narrow. Without signatures,
  any workload with produce rights on the spine can forge `producer: <other-agent>`.
  Kafka ACLs per principal (one credential per agent runtime, ACL restricting which
  `producer` values / topics it may write) close most of this **without any envelope
  crypto** — the broker becomes the authenticator. Signature adds value only against an
  attacker who has the broker or a shared credential, i.e., mostly the insider/full-host
  compromise case where they likely also hold the signing key.
- **Tamper-evidence at rest** — PARTIAL. The Kafka log is append-only in operation but
  not cryptographically so; an admin (or ransomware with admin creds) can rewrite
  segments/re-produce a doctored topic. Per-event signatures make *individual events*
  tamper-evident but not *the sequence* (deletion/reordering needs a hash chain or
  signed head). Buzz's own caveat applies verbatim: tamper-evident, not tamper-resistant.
- **Non-repudiation** — MOSTLY THEATER here. Repudiation matters between mutually
  distrusting parties. Greenwood R1 has one company, one operator, agents that are not
  legal persons, and keys provisioned by the same runtime that would be "repudiating."
  A signature by a key the operator minted and stored proves nothing to a third party.
- **Audit quality** — REAL, and the honest core of the Buzz idea. "Every action
  attributable to exactly one agent identity" is an *identity discipline*, not a
  cryptography requirement. Stable per-agent IDs + per-agent broker credentials +
  per-agent API keys give the same audit answer at R1 scale; signatures upgrade the
  audit from "trusted because we ran it" to "checkable later" only once there's a
  second party (customer, auditor, federated deployment) to check it.

**Where signing is real security for Greenwood:** (a) if bus events are ever exported /
shared outside the cluster (compliance evidence, cross-org replay); (b) the
originating-human field — integrity protection (even HMAC by the gateway) is what makes
"no delegated action exceeds the human's tier" enforceable against a lying intermediate
agent; (c) approval events (`ApprovalGranted`) — signing the *human decision* is worth
more than signing every agent thought.

**Where it is theater at R1:** signing every `thought`/`stream_delta` with per-agent
Ed25519 keys held on the same host as the runtime, verified by the same operator's bus.
The threat it addresses (operator-forged history) is not addressed, because the operator
holds the keys.

## Recommendation — envelope fields to reserve day one vs. defer

**Reserve in the T1.1 Protobuf on day one (cheap: reserved field numbers + one message
type), implement almost none of it for R1:**

1. **Structure `producer` into a principal.** Replace bare `string producer` with (or
   add alongside, keeping 4 as deprecated alias):
   ```proto
   message Principal {
     string id       = 1;  // stable agent id — use as gen_ai.agent.id (DC-4 alignment)
     string kind     = 2;  // agent | human | bus | verifier | runtime
     string runtime  = 3;  // which harness/host asserted this identity
   }
   Principal producer_principal = 8;
   ```
   R1 behavior: runtime-asserted, transport-authenticated (per-agent Kafka principal +
   ACLs). This is the identity-discipline core of the Buzz idea with zero crypto.
2. **`OriginatingHuman originating_human = 9`** (DC-3, already a decision candidate):
   `{human_id, role, auth_strength, session_id, mac}` — with the `mac` field (HMAC by
   the gateway/bus over the other four) *reserved now, populated when the gate ships*.
   This is the one signature-like field with a real R1 threat behind it.
3. **`Signature signature = 23`** reserved, not populated (note: the payload oneof's
   members occupy the enclosing message's field-number space 10-22, so new top-level
   fields must use 8, 9, or 23+):
   ```proto
   message Signature {
     string key_id   = 1;  // key registry reference, supports rotation
     string scheme   = 2;  // "ed25519-over-event_id" etc.
     bytes  sig      = 3;  // over event_id — canonical-hash spec is the message
   }
   ```
   Because `event_id` is already a canonical content hash with cross-language test
   vectors, retro-fitting signatures later is *only* a key-management project, not a
   schema/canonicalization migration — provided the field number is reserved and the
   rule "signature covers event_id" is written down now.
4. **One acceptance-criteria addition to T1.1:** the canonical-hash spec is declared
   the signing pre-image; any future signature signs `event_id`, never ad-hoc bytes.

**Defer (post-R1, in order of likely pull):** per-agent Kafka principals + broker ACLs
binding principal→producer_principal.id (M1-M2, first real anti-impersonation control);
HMAC on originating_human (with the gate work); signed approval/policy-decision events
(sign the human decisions first — highest audit value per key); lineage-segment
attestation (Merkle root of event_ids signed at session close — Sigstore/in-toto shape)
if compliance export ever appears; SPIFFE/SPIRE, per-agent keypairs, per-event
signatures — only if Greenwood ever federates across trust domains (the Buzz-shaped
world Greenwood is not in).

**Do NOT adopt:** Nostr-style keypair-per-agent with signature-per-event for R1. In a
single-cluster deployment where the operator provisions the keys, it is audit theater
that costs real key-management surface. Buzz needs it because its relay is untrusted;
Greenwood's bus is the trust anchor.

## Sources

**Method note (2026-07-29):** live WebSearch/WebFetch were unavailable for most of this
pass (tool-gating classifier outage). Findings are grounded in (a) URLs already
verified/reported in this repo's pass-1 and deep-dive notes (carried with their
original verification tags), (b) stable pre-2026 specs cited from training knowledge
and flagged as such, (c) local repo analysis. Items marked [could not re-verify live]
should get a URL check next web session; none of them are load-bearing for the
envelope recommendation.

Carried from local notes (original verification in brackets):
- https://www.hashicorp.com/en/blog/spiffe-securing-the-identity-of-agentic-ai-and-non-human-actors [reported, pass-1]
- https://www.paloaltonetworks.com/blog/identity-security/ai-agent-security-spiffe-machine-identity/ [reported, pass-1]
- https://nhimg.org/articles/spiffe-and-ai-agent-identity-expose-the-next-authorization-gap/ [reported, pass-1]
- https://labs.cloudsecurityalliance.org/agentic/agentic-identity-governance-framework-v1/ [reported, pass-1]
- https://www.digitalapplied.com/blog/agent-identity-credentials-non-human-access-2026-playbook [reported, pass-1; NIST NCCoE claim unverified against nist.gov]
- https://aws.amazon.com/blogs/security/enforce-least-privilege-authorization-in-multi-agent-ai-chains-using-cedar/ [verified, gates deep dive] — ASI03 + HMAC user context
- https://developer.microsoft.com/blog/securing-mcp-a-control-plane-for-agent-tool-execution/ [verified, gates deep dive] — SPIFFE-compatible agent IDs, hash-chained audit
- https://developer.hashicorp.com/validated-patterns/vault/ai-agent-identity-with-hashicorp-vault [verified, gates deep dive]
- https://www.sans.org/blog/your-ai-agent-easily-confused-deputy-why-cloud-security-needs-credential-broker [verified, pass-1] — CB4A broker, DPoP-bound per-operation tokens
- https://blog.modelcontextprotocol.io/posts/2026-07-28/ [verified, gates deep dive] — MCP leaves identity/audit to the runtime layer
- https://rohitraj.tech/en/notes/block-buzz-agent-collaboration-platform-guide-2026 ; https://decrypt.co/374026/jack-dorseys-block-launches-buzz-a-nostr-based-slack-and-github-rival-for-ai-agents [verified, competitor-hive] — Buzz keypair-per-agent, NIP-01/NIP-42
- https://opentelemetry.io/docs/specs/semconv/registry/attributes/gen-ai/ [verified, cost deep dive] — `gen_ai.agent.id` et al.
- https://platform.claude.com/docs/en/manage-claude/usage-cost-api [verified, cost deep dive] — key-per-agent attribution axis

Stable specs cited from training knowledge (not re-fetched today):
- https://github.com/nostr-protocol/nips/blob/master/01.md (NIP-01)
- https://spiffe.io/docs/latest/spiffe-about/overview/ (SPIFFE/SVID)
- https://www.sigstore.dev/ ; https://github.com/in-toto/attestation
- https://datatracker.ietf.org/doc/html/rfc8693 (token exchange) ; https://datatracker.ietf.org/wg/wimse/about/
- https://owasp.org/www-project-non-human-identities-top-10/
- https://modelcontextprotocol.io/specification/2025-06-18/basic/authorization (MCP OAuth 2.1 authorization)
- https://c2pa.org/

Open to verify next web session: current status of IETF/OAuth agent-specific profiles
(WIMSE deliverables, any "OAuth for AI agents" draft adoption); whether the NIST NCCoE
AI-agent-identity concept paper is published on nist.gov; SPIFFE community
AI-agent-pattern posts newer than the two vendor pieces above.
