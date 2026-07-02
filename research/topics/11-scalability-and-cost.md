---
doc_type: design-sketch
topic: Scalability & cost — is Kafka right, cheaper backings, and a 500-agent-year cost model
project: Greenwood
tags: [scalability, cost, kafka, warpstream, automq, s3, tiered-storage, eval-cost, snapshots, payloads-by-reference]
sources: [research-agents, engineering]
created: 2026-07-01
updated: 2026-07-01
status: draft
summary: Greenwood's workload is write-heavy with rare/cold reads. Vanilla Kafka on triple-replicated EBS + cross-AZ replication is the wrong cost model for it. Keep the Kafka API but run it on an object-store-native engine (AutoMQ / WarpStream) → storage ≈ single-copy S3, no inter-AZ tax. At 500 agents Greenwood is a SMALL throughput workload (sub-10 MB/s) — storage- and compute-floor-bound, not throughput-bound. State capture/replay ≈ low tens of $k/yr. The eval (Coldframe) bucket is LLM-inference-dominated and dwarfs storage.
---

# Scalability & cost

Research: three parallel agents (scalability profile · Kafka alternatives · prior art).
Prices are US list, mid-2026, undiscounted; see the cited agent briefs for sources.

## Is Kafka the right choice?
The **Kafka abstraction** is right — append-only log, key-based ordering, replay from
offset, compaction for projections, tiered retention. **Vanilla open-source Kafka is the
wrong way to run it** for this workload. Greenwood is write-heavy with rare, cold reads;
vanilla Kafka's bill is dominated by the two things that hurt that shape most:
1. **Hot storage × replication factor** — every event on 3× replicated SSD for the whole
   hot window.
2. **Inter-AZ replication traffic** — ~$0.04 per GB ingested at RF=3 multi-AZ (the leader
   ships each GB to 2 followers; each cross-AZ flow bills $0.01 out **+** $0.01 in), and
   that's before the producer→leader leg (~⅔ cross-AZ on average, ~+$0.013/GB). Cross-AZ
   traffic can be >50% of a Kafka bill (independently corroborated).
Read/egress — normally the expensive axis — is ~$0 here. So we're paying Kafka's most
expensive dimensions and idling its cheap one.

**Decision (see D6):** keep the Kafka API, run it on an **object-store-native, Kafka-
compatible engine** — **AutoMQ** (Kafka fork: full compaction via Kafka's real LogCleaner;
sub-10ms P99 with a single-AZ EBS WAL, ~20ms multi-AZ; Apache 2.0 since Apr 2025; drop-in)
or **WarpStream** (managed, drop-in; watch compaction ceilings: 3M distinct keys /
128 GiB per job, immutable `cleanup.policy`; owned by Confluent, whose acquisition by
IBM closed 2026-03-17 — a vendor-risk line item). These drop triple-replicated NVMe for
S3 and eliminate cross-AZ replication → storage/network ~5–10× cheaper while preserving
every property (and the design + Graft adapters unchanged). A further vanilla-only
constraint: **KIP-405 tiered storage does not support compacted topics**, so on vanilla
Kafka the compacted projections (`confirmed`, lineage) must stay on local disk — only the
raw log tiers to S3. AutoMQ sidesteps this (compaction runs on its S3-backed storage),
which strengthens the S3-native case. **Raw S3-as-log** is cheapest at scale but costs a
full rewrite (you own ordering/index/fencing) — fallback, not default. Avoid:
Kinesis/Pub/Sub (no compaction, retention ceilings of 365d/31d); NATS JetStream (no
cold/object-store tier — open requests nats-io #3871/#5486 — and only crude per-subject
last-N retention, not Kafka-style compaction); Pulsar (topic compaction fails on S3
tiered storage — apache/pulsar#8282, unfixed, closed as stale).

## Three byte-shrinking levers (engine-independent, from prior art)
- **Payloads by reference** — large tool outputs / model responses go to an S3 blob +
  a hash/pointer in the event. The ingest-vs-store gap is huge — derived from W&B Weave's
  pricing, $0.10/MB one-time ingest vs $0.03/GB·mo storage ≈ 3,300× (per GB, vs a month
  stored) — so keeping payloads out of the log keeps the avg event ~1 KB no matter how
  big a tool result is.
- **Snapshot to bound hot retention** — periodic state snapshot to object storage; resume
  replays only the tail since the snapshot. Lets the hot window be hours, not days.
  (Pure replay-from-genesis has a hard ceiling — Temporal hard-terminates at 51,200
  events / 50 MB of history; Cadence warns there and kills at ~205K/200 MB.)
- **Cold tier to S3-IA / Glacier** — the multi-year archive is the balloon by *volume*;
  at PB scale S3 Standard is wasteful. Rare eval-replay reads make IA/Glacier fine.

## Cost model — state capture / replay (the 500-agent-year calc)

**Scope:** state capture + replay only (ignore agent-runtime inference). Driving parameter
is **captured bytes per agent-second** (events + referenced-payload bytes).

**Token-native basis (interactive calculator, `research/cost-model/`):** since agent
utilization is usually reasoned about in *tokens*, the calculator takes *total net-new
captured tokens / agent·hour* — model output **plus tool calls, tool results, and claims**
(each logged once; context re-sends excluded), which for a busy agent is dominated by tool
results, not the model's own output (default ~600k/hr ≈ 167 tok/s ≈ 1 KB/s — the "Low"
bracket below; tool-heavy agents hit 1M+). It converts at ~4 B/token × ~1.5 envelope overhead. It also prices **S3 GET
requests** (replay reads), which — like PUTs — scale with **segment size, not event
count**: batched MB-scale segments keep request cost negligible; one-object-per-event
would make requests explode. At realistic net-new token rates the capture volume comes out
at or below the byte estimates below — reinforcing that storage is trivial and eval
inference dominates. Compute is sized **bottom-up** — `nodes = ceil(throughput · RF ÷
per-node capacity)`, floored at an HA minimum — so it stays a flat floor at the 500-agent
scale but grows linearly once the fleet is large enough to exceed a node's capacity (the
calculator scales to arbitrary fleet size; it defaults to a 500k-agent fleet).

| | captured B/agent/s | throughput (500 agents) | annual volume V |
|---|---|---|---|
| Low | ~1 KB/s | 0.5 MB/s | ~16 TB |
| Mid | ~5 KB/s | 2.5 MB/s | ~79 TB |
| High | ~20 KB/s | 10 MB/s | ~316 TB |

**Key structural facts at this scale:**
- Throughput (≤10 MB/s) is **trivial** for any engine — a minimal cluster handles it. So
  **compute is a near-fixed floor** (~$1–3k/mo managed S3-native, less self-hosted), not
  throughput-driven.
- **Storage dominates over time**, and it *accumulates* over the year (steady-state ≈ V
  fully retained; year-1 ≈ ½V average).
- Egress ≈ $0 (cold reads).

**Annual cost, Mid case (79 TB), by approach:**
| Approach | Storage | Compute | Network | ≈ Total/yr |
|---|---|---|---|---|
| Vanilla Kafka, **all hot**, RF=3, gp3 | 79 TB×3×$0.08×12 ≈ **$2.27M** | small | +inter-AZ | **non-starter** |
| Vanilla Kafka, tiered (short hot + S3 Std cold)¹ | ~$22k (S3 Std, 1×) | ~$12–36k | inter-AZ on ingest ~$3.2k | **~$40–60k** |
| **S3-native (AutoMQ/WarpStream)** | ~$22k S3 Std / ~$12k IA / ~$3k Glacier | ~$12–36k (+license) | ~$0 cross-AZ | **~$25–60k** |
| Raw S3-as-log | ~$12–22k + ~$2k PUT | ~$6–12k | ~$0 | **~$20–35k** |

¹ Vanilla-tiered caveat: KIP-405 excludes compacted topics, so the compacted projections
stay on replicated local disk (small volume, but not $0), and the raw log alone tiers.

So **a year of state capture for 500 agents lands in the low tens of $k** (mid
assumptions), dominated by the compute floor + accumulated S3 storage — *not* by
throughput. Low case ≈ $15–30k; high case (316 TB) ≈ $60–120k. Cold-tiering the archive
(IA/Glacier) roughly halves-to-quarters the storage line.

**Formula:** `V = 500 · B · 3.156e7` (bytes/yr); `storage$/yr ≈ V_GB · rate$/GB-mo · 12`
(× ½ in year 1); `compute$/yr ≈ max(HA-floor, nodes·$/node)`; `network ≈ 0` on S3-native. Vanilla-Kafka
adds `inter-AZ = V_GB · $0.04` (2 follower streams × $0.02 round trip) and
`hot = hotwindow · V · RF · $0.08/GB-mo`.

## Cost model — evals (Coldframe): LLM-inference-dominated
Evals re-run the agentic flow through the model, so this bucket is **inference**, not
storage — and it typically **dwarfs** the capture bucket.

`eval$/yr = suite_cases · runs_per_case · cost_per_run`, where `runs_per_case = N (samples
for pass-rate) · iterations_to_pass`, and

`cost_per_run ≈ ctx_tokens · input_rate + out_tokens · output_rate`.

`claude-opus-4-8`: input $5/M, output $25/M, **cache-read ~$0.50/M** (0.1×), cache-write
$6.25/M (5-min TTL; $10/M for 1-hour). Hermetic replay reconstructs the frozen prefix
identically → the context bills at cache rates, and recorded tool-results as fixtures
mean **no tool-execution cost**. The cache TTL matters: each case's **first sample per
re-run writes the cache** (1.25× input); only the other N−1 samples read it — so the
amortized context rate is `(write + (N−1)·reads)/N`, not pure cache-read.

Per-run (50K ctx + 2K out): **cached ≈ $0.17 amortized** (N=3; $0.13 at N=5), **cold ≈
$0.30**.

| suite | re-runs/yr | N | runs/yr | cached (amortized) | @ $0.30 (cold) |
|---|---|---|---|---|---|
| 1,000 | 52 (weekly) | 3 | 156k | ~$27k | ~$47k |
| 10,000 | 52 | 5 | 2.6M | ~$345k | ~$780k |

So evals range from **~$25k to ~$800k+/yr** purely on cadence — and at any real regression
cadence, **eval inference is the dominant Greenwood cost**, not event storage. Levers:
**eval selection** (re-run only the affected subset per harness change, not the whole
suite), keep N small (but note N=1 forfeits the cache discount entirely — every run is a
write), batch a case's N samples inside one TTL window, use a cheaper judge model.

Related TTL economics for the *runtime* (not evals): a blanket `ttl:"1h"` policy costs a
$3.75/M premium on every net-new prefix token ≈ **$2.25/agent·hr always-on**, vs ≈ $0.90
saved per crash-resume — so TTL is a per-agent cadence policy, 5-min default (topic 06).

## Cost model — governance & handoff inference (the third bucket)
The bucket the first two hide: inference that exists **only because of Greenwood** —
claim decomposition (structured output on every turn), Grieve verification (an LLM step
per verified claim), and the per-handoff gate (on-demand slice verification + Tier-0
synthesis + entailment check, ×2 checkers for high-risk). Roughly:

`gate$/handoff ≈ unverified_slice_claims·verify$ + (slice_ctx+synth_out)·rates + entail$`
`verify$/claim ≈ (claim + evidence_ref ctx)·input_rate + verdict_out·output_rate`

At plausible rates (a few cents per verification, dimes per handoff) this bucket scales
with *agent activity* — unlike evals it can't be batched off-peak, and it sits in the
handoff critical path (latency AND dollars). At fleet scale it plausibly **rivals the
eval bucket**; decomposition overhead alone is ~5–15% extra output tokens on all agent
traffic. **The funding-level ROI question** (record it, don't bury it): is Greenwood's
added inference overhead paid back by avoided cascade-redo — i.e., does gated handoff +
targeted rewind save more re-derivation inference than the governance itself costs?
Instrument this from day one (redo-tokens avoided vs governance-tokens spent).

## Open parameters to pin down
1. **Captured bytes/agent-second** — the driver of the storage bucket; measure the real
   event-rate and payload-size distribution (with payloads-by-reference in place).
2. **Hot-retention window & snapshot cadence** — sets how little sits on expensive hot
   storage.
3. **Cold archive class & duration** — S3 Std vs IA vs Glacier for the multi-year tail.
4. **Eval cadence, suite size, N, selection policy** — dominates total cost; decide the
   regression-run policy deliberately.
5. **Engine choice** — AutoMQ vs WarpStream vs raw-S3; measure S3-path produce latency.
   (Compaction-vs-cardinality is now answered: the claim keyspace is monotone, so
   compacted claim state needs TTL-after-terminal-state or a DB-backed projection at
   fleet scale — see topic 08 §2 and D6.)
