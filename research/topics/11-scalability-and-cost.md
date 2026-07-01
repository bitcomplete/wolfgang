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
2. **Inter-AZ replication traffic** — ~$0.02 per GB ingested at RF=3 multi-AZ; can be
   >50% of a Kafka bill (independently corroborated).
Read/egress — normally the expensive axis — is ~$0 here. So we're paying Kafka's most
expensive dimensions and idling its cheap one.

**Decision (see D6):** keep the Kafka API, run it on an **object-store-native, Kafka-
compatible engine** — **AutoMQ** (full compaction via Kafka's real LogCleaner, ~20ms P99
with an EBS WAL, drop-in) or **WarpStream** (managed, drop-in; watch compaction ceilings:
3M distinct keys / 128 GiB per job, immutable `cleanup.policy`). These drop triple-
replicated NVMe for S3 and eliminate cross-AZ replication → storage/network ~5–10× cheaper
while preserving every property (and the design + Graft adapters unchanged). **Raw
S3-as-log** is cheapest at scale but costs a full rewrite (you own ordering/index/fencing)
— fallback, not default. Avoid: Kinesis/Pub/Sub/JetStream (no compaction and/or no cheap
cold tier and/or full rewrite); Pulsar (compaction-on-tiered-data is a known bug).

## Three byte-shrinking levers (engine-independent, from prior art)
- **Payloads by reference** — large tool outputs / model responses go to an S3 blob +
  a hash/pointer in the event. Ingest-vs-store cost gap is ~3,000× (W&B Weave), so keeping
  payloads out of the log keeps avg event ~1 KB no matter how big a tool result is.
- **Snapshot to bound hot retention** — periodic state snapshot to object storage; resume
  replays only the tail since the snapshot. Lets the hot window be hours, not days.
  (Pure replay-from-genesis has a hard ceiling — Temporal/Cadence wall at ~50K events.)
- **Cold tier to S3-IA / Glacier** — the multi-year archive is the balloon by *volume*;
  at PB scale S3 Standard is wasteful. Rare eval-replay reads make IA/Glacier fine.

## Cost model — state capture / replay (the 500-agent-year calc)

**Scope:** state capture + replay only (ignore agent-runtime inference). Driving parameter
is **captured bytes per agent-second** (events + referenced-payload bytes).

**Token-native basis (interactive calculator, `research/cost-model/`):** since agent
utilization is usually reasoned about in *tokens*, the calculator takes *output tokens /
agent·hour* and converts at ~4 B/token × ~1.5 envelope overhead. It also prices **S3 GET
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
| Vanilla Kafka, tiered (short hot + S3 Std cold) | ~$22k (S3 Std, 1×) | ~$12–36k | inter-AZ on ingest ~$1.6k | **~$40–60k** |
| **S3-native (AutoMQ/WarpStream)** | ~$22k S3 Std / ~$12k IA / ~$3k Glacier | ~$12–36k (+license) | ~$0 cross-AZ | **~$25–60k** |
| Raw S3-as-log | ~$12–22k + ~$2k PUT | ~$6–12k | ~$0 | **~$20–35k** |

So **a year of state capture for 500 agents lands in the low tens of $k** (mid
assumptions), dominated by the compute floor + accumulated S3 storage — *not* by
throughput. Low case ≈ $15–30k; high case (316 TB) ≈ $60–120k. Cold-tiering the archive
(IA/Glacier) roughly halves-to-quarters the storage line.

**Formula:** `V = 500 · B · 3.156e7` (bytes/yr); `storage$/yr ≈ V_GB · rate$/GB-mo · 12`
(× ½ in year 1); `compute$/yr ≈ max(HA-floor, nodes·$/node)`; `network ≈ 0` on S3-native. Vanilla-Kafka
adds `inter-AZ = V_GB · $0.02` and `hot = hotwindow · V · RF · $0.08/GB-mo`.

## Cost model — evals (Coldframe): LLM-inference-dominated
Evals re-run the agentic flow through the model, so this bucket is **inference**, not
storage — and it typically **dwarfs** the capture bucket.

`eval$/yr = suite_cases · runs_per_case · cost_per_run`, where `runs_per_case = N (samples
for pass-rate) · iterations_to_pass`, and

`cost_per_run ≈ ctx_tokens · input_rate + out_tokens · output_rate`.

`claude-opus-4-8`: input $5/M, output $25/M, **cache-read ~$0.50/M** (0.1×). Hermetic
replay reconstructs the frozen prefix identically → the large context bills at cache-read,
and recorded tool-results as fixtures mean **no tool-execution cost**.

Per-run (50K-token context + 2K output): **cache-hit ≈ $0.08**, **cold ≈ $0.30** (4×).

| suite | re-runs/yr | N | runs/yr | @ $0.08 (cached) | @ $0.30 (cold) |
|---|---|---|---|---|---|
| 1,000 | 52 (weekly) | 3 | 156k | ~$12k | ~$47k |
| 10,000 | 52 | 5 | 2.6M | ~$208k | ~$780k |

So evals range from **~$10k to ~$800k+/yr** purely on cadence — and at any real regression
cadence, **eval inference is the dominant Greenwood cost**, not event storage. Levers:
**eval selection** (re-run only the affected subset per harness change, not the whole
suite), keep N small, use a cheaper judge model, and lean on the cache-read discount.

## Open parameters to pin down
1. **Captured bytes/agent-second** — the driver of the storage bucket; measure the real
   event-rate and payload-size distribution (with payloads-by-reference in place).
2. **Hot-retention window & snapshot cadence** — sets how little sits on expensive hot
   storage.
3. **Cold archive class & duration** — S3 Std vs IA vs Glacier for the multi-year tail.
4. **Eval cadence, suite size, N, selection policy** — dominates total cost; decide the
   regression-run policy deliberately.
5. **Engine choice** — AutoMQ vs WarpStream vs raw-S3; verify compaction against real
   `lineage_root` cardinality and measure S3-path produce latency.
