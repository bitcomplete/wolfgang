---
title: Fireworks.ai serverless inference pricing — August 2026 snapshot
date: 2026-08-03
topic: LLM inference cost — Fireworks serverless per-token pricing, caching, batch, rate limits
status: draft
---

# Fireworks.ai serverless pricing — verified 2026-08-03

Primary source: https://docs.fireworks.ai/serverless/pricing (fireworks.ai/pricing itself no
longer lists per-token text-model prices; it points to the docs page). Prices are the
**Standard serverless tier** unless noted; format is $/M tokens.

## 1. Strongest coding/agentic models (Standard tier)

| Model | Input $/M | Cached input $/M | Output $/M | Tag / seen |
|---|---|---|---|---|
| Kimi K3 | 3.00 | 0.30 | 15.00 | verified 2026-08-03 |
| Kimi K2.7 Code | 0.95 | 0.19 | 4.00 | verified 2026-08-03 |
| Kimi K2.6 | 0.95 | 0.16 | 4.00 | verified 2026-08-03 |
| DeepSeek V4 Pro | 1.74 | 0.145 | 3.48 | verified 2026-08-03 |
| GLM 5.2 | 1.40 | 0.14 | 4.40 | verified 2026-08-03 |
| GLM 5.1 | 1.40 | 0.26 | 4.40 | verified 2026-08-03 |
| Qwen 3.7 Plus | 0.40 | 0.08 | 1.60 | verified 2026-08-03 |
| MiniMax M3 | 0.30 | 0.06 | 1.20 | verified 2026-08-03 |
| MiniMax M2.7 | 0.30 | 0.06 | 1.20 | verified 2026-08-03 |
| NVIDIA Nemotron 3 Ultra (Preview) | 0.60 | 0.12 | 2.40 | verified 2026-08-03 |

Tier variants (same docs page, verified 2026-08-03):
- **Priority tier** ≈ 1.2–1.5x Standard (e.g. Kimi K3 Priority 3.75/18.75; GLM 5.2 Priority
  1.75/5.50; DeepSeek V4 Pro Priority 2.61/5.22). Priority buys lower queueing, same rate-limit pool.
- **Fast tier** ≈ 1.5–2x Standard for 100+ tok/s targets (Kimi K3 Fast 4.50/22.50;
  Kimi K2.6 Fast 2.00/8.00; GLM 5.2 Fast 2.10/6.60; GLM 5.1 Fast 2.80/8.80;
  Kimi K2.7 Code Fast 1.90/8.00).
- **US-only endpoints** cost +10% (e.g. Kimi K3 US 3.30/16.50) — exception: GLM 5.2 Fast US
  matches global pricing.

Notes:
- Qwen3 Coder 480B A35B is no longer serverless on Fireworks — on-demand GPU deployment only
  (https://fireworks.ai/models/fireworks/qwen3-coder-480b-a35b-instruct). Qwen 3.7 Plus is the
  current serverless Qwen line. (verified 2026-08-03)
- No Llama 4.x row appeared among the headline priced models on the docs pricing page; generic
  parameter-count pricing would apply if deployed (estimate).
- Generic size-based serverless pricing (verified 2026-08-03): <4B $0.10/M; 4–16B $0.20/M;
  >16B $0.90/M; MoE ≤56B $0.50/M; MoE 56.1–176B $1.20/M (single rate, input+output).

## 2. Prompt caching

- **Automatic, on by default** for all models and deployments; Fireworks matches the longest
  cached prefix automatically — no API opt-in needed. (verified 2026-08-03,
  https://docs.fireworks.ai/guides/prompt-caching)
- Default cached-input discount is **50%**, but per-model rates vary and are often much better —
  e.g. Kimi K2.6 cached input $0.16 vs $0.95 (~83% off); DeepSeek V4 Pro $0.145 vs $1.74
  (~92% off); GLM 5.2 $0.14 vs $1.40 (90% off). See table above. (verified 2026-08-03)
- Cache TTL: "usually at least several minutes… up to several hours" depending on model/load.
  (verified 2026-08-03)

## 3. Batch / off-peak

- **Batch inference is billed at 50% of serverless pricing** (inputs and outputs).
  (verified 2026-08-03, https://docs.fireworks.ai/serverless/pricing)
- No off-peak/time-of-day discount found. (verified 2026-08-03, absence of evidence)

## 4. Small/cheap screening models

| Model | Input $/M | Cached input $/M | Output $/M | Tag / seen |
|---|---|---|---|---|
| DeepSeek V4 Flash | 0.14 | 0.028 | 0.28 | verified 2026-08-03 |
| DeepSeek V4 Flash (0731) | 0.14 | 0.028 | 0.28 | verified 2026-08-03 |
| GPT OSS 20B | 0.07 | 0.035 | 0.30 | verified 2026-08-03 |
| GPT OSS 120B | 0.15 | 0.015 | 0.60 | verified 2026-08-03 |

Recommendation for triage: **DeepSeek V4 Flash** ($0.14/$0.28) — strong general small model,
deep cache discount; GPT OSS 120B is a close alternative with near-free cached input ($0.015).

## 5. Fees, rate limits, tiers

- **No per-request fees** disclosed anywhere on the pricing docs. (verified 2026-08-03)
- Account-wide request ceiling: **10 RPM with no payment method; 6,000 RPM fixed ceiling with
  card + credits** (applies across serverless + deployments + fine-tuning).
  (verified 2026-08-03, https://docs.fireworks.ai/guides/quotas_usage/rate-limits)
- Serverless TPM limits are **adaptive, per account per model**. Default ceilings:
  **21.6M total prompt TPM, 5.4M uncached prompt TPM, 216k generated TPM**. Limits ramp with
  usage; rapid traffic spikes can 429 even under the nominal ceiling while the adaptive ramp
  catches up. Fast vs regular variants have independent pools; Priority shares the standard
  pool. (verified 2026-08-03, https://docs.fireworks.ai/serverless/rate-limits)
- **Spending tiers** gate max monthly budget and TPM upper bounds: Tier 1 $50/mo (valid card),
  Tier 2 $500/mo, Tier 3 $5,000/mo, Tier 4 $50,000/mo; higher tier → higher rate-limit upper
  bounds; enterprise gets more automatically. (estimate — tier dollar figures via secondary
  sources cross-checked against docs statement that tiers control budget + TPM bounds; seen
  2026-08-03)
- The 216k generated-TPM default ceiling is the binding constraint for sustained agent
  workloads (output-heavy); email inquiries@fireworks.ai for raises beyond adaptive/tier
  maximums. (verified 2026-08-03)

## Sources

- https://docs.fireworks.ai/serverless/pricing — per-token prices, tiers, batch 50%
- https://fireworks.ai/pricing — top-level page (redirects text-model pricing to docs)
- https://docs.fireworks.ai/guides/prompt-caching — caching behavior + discount
- https://docs.fireworks.ai/guides/quotas_usage/rate-limits — account RPM, spending tiers
- https://docs.fireworks.ai/serverless/rate-limits — adaptive TPM ceilings
- https://fireworks.ai/models/fireworks/qwen3-coder-480b-a35b-instruct — Qwen3 Coder 480B not serverless
