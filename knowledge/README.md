# Wolfgang Knowledge Base — index

**What this KB covers:** the Software Garden — Bit Complete's system for agents
operating software autonomously with humans at meaning boundaries. First release
(R1) ships **2026-08-16**.

**Retrieval map** (one concept per file; start at the overview):

| File | Answers questions about |
|---|---|
| `garden/00-overview.md` | What are we building, for whom, by when, under what constraints |
| `garden/01-autonomy-model.md` | Autopilot vs human gates; rings/environments; app modes (standardize/build/run); human engagement surfaces |
| `garden/02-trust-attestations.md` | Catching hallucinations; attestations & evidence ledger; approvals; incident/break-glass |
| `garden/03-substrate-event-log.md` | Event sourcing; the swappable log (Postgres, not Kafka); envelope schema |
| `garden/04-feedback-learning.md` | How the system learns; feedback capture & propagation; eventual consistency |
| `garden/05-operator-surface.md` | Cards, runbooks, staleness honesty, patrols, digests |
| `garden/06-decisions-index.md` | Every decision (P0, R1, D1–D14) one line each, with status |
| `garden/07-open-questions.md` | What is NOT decided yet |
| `ARCHITECTURE.md` | Greenwood **reference architecture** (the full Kafka-era spec — idea source per D11, not the deliverable) |

**Conventions:**
- Every page carries `created`/`updated` dates and a `status` line — check them; do
  not answer authoritatively from stale pages (that failure mode is documented:
  `research/topics/12-garden-hygiene-antipatterns.md` §1).
- **DECIDED** claims cite a decision id (e.g. D13) with its date. Anything not marked
  decided is proposal or open — never present it as settled.
- Deep provenance lives in `../research/` — `decisions.md` (the ADR log, full
  rationale), `R1-WORK-BREAKDOWN.md` (active ticket source),
  `software-garden-2026-07/` (the 2026-07-29 research sweep: 21 notes, synthesis,
  4 reports, `r1-spec-inputs.md` with DC-1..13).
