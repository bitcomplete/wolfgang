---
title: "Round-2 verification: headline statistics against primary sources"
date: 2026-07-29
topic: primary-source verification of 12 statistics cited in software-garden spec discussions
status: draft
---

# Round-2 stat verification (primary sources)

**Method:** each claim was checked against its primary source on 2026-07-29 (arXiv page fetch, first-party post, or the repo's own full-paper review). **Caveat on completeness:** a multi-hour outage of the tool-safety classifier blocked all web access (WebFetch/WebSearch/curl/subagents) partway through this pass; four arXiv abstracts were fetched live before the outage, one claim is settled by a full-paper review already in this repo, and two claims rest on model knowledge of pre-2026 primary sources (flagged). Everything else is marked **BLOCKED** and must be re-run when web access recovers — BLOCKED means *not yet checked*, not *false*.

Verdicts: **VERIFIED** (exact figure quoted from primary), **CORRECTED** (real figure/claim differs), **UNVERIFIABLE** (no primary source exists — do not cite), **BLOCKED** (outage prevented the check — do not cite until re-run).

## Verdict table

| # | Claim | Verdict | Exact primary figure | Source |
|---|-------|---------|---------------------|--------|
| 1 | Spark-to-Fire ≥89% cascade prevention | VERIFIED (live fetch + local full-paper review) | "prevents final infection in at least 89% of runs across operating modes" — Speed ~0.89 / Balanced 0.93 / Strict 0.94 BICR vs ≈2.2% no-defense, ≈3.1% detection-only | https://arxiv.org/abs/2603.04474 |
| 2 | SREGym 38.9–72.6% diagnosis | PARTIALLY VERIFIED (live abstract fetch) | Abstract: 90 realistic SRE problems; "up to 40% differences in end-to-end results" across failure kinds. The 38.9–72.6% / 57.3–78.5% ranges and model names are NOT in the abstract — full-text check blocked | https://arxiv.org/abs/2605.07161 |
| 3 | MAST 1,600+ traces; MAS gains minimal | VERIFIED (live fetch); affiliation note CORRECTED | "1600+ annotated traces collected across 7 popular MAS frameworks"; gains "often minimal"; 14 modes / 3 categories. Neubig is NOT a MAST author — it's the UC Berkeley group (Cemri … Stoica) | https://arxiv.org/abs/2503.13657 |
| 4 | Anthropic ~15× token cost | VERIFIED (pre-cutoff primary; live re-check blocked) | Post says agents use ~4× more tokens than chat, and "multi-agent systems use about 15× more tokens than chats"; token usage explains ~80% of performance variance on BrowseComp | https://www.anthropic.com/engineering/built-multi-agent-research-system |
| 5 | 2602.00496 junior over-trust | CORRECTED (live fetch) | Abstract: "novices struggle between over-reliance and cautious avoidance" — mis-calibration in both directions, small-N qualitative (10 juniors, 10 seniors) | https://arxiv.org/abs/2602.00496 |
| 6 | METR 19% slower / believed 20% faster | VERIFIED (pre-cutoff primary; live re-check blocked) | 16 experienced OSS devs, 246 tasks: with AI allowed, completion took 19% longer; post-study, participants estimated AI had sped them up ~20% (pre-study forecast ~24%) | https://metr.org/blog/2025-07-10-early-2025-ai-experienced-os-dev-study/ |
| 7 | Anthropic 47% debugging-skill drop | UNVERIFIABLE — do not cite | No primary Anthropic source found in round 1 (secondary blogs only) and none exists in model knowledge through Jan 2026; live confirmation sweep blocked | none found |
| 8 | $47k LangChain loop; InfoQ $14k / $6.5k incidents | $47k: UNVERIFIABLE — do not cite. InfoQ pair: VERIFIED in round 1; re-check blocked | $47k appears only in two dev.to posts, no named company. InfoQ (July 2026): $14,000/day leaked-key Bedrock incident; DN42 autonomous-provisioning $6,531.30 (AWS negotiated to $1,894) | https://www.infoq.com/news/2026/07/ai-agents-billing-guardrails/ |
| 9 | >4h agents ~90% higher total-failure risk | UNVERIFIABLE — do not cite (origin check blocked) | Round-1 already flags it as an unverified vendor number from industry-blog-grade sources (slavadubrov.github.io / appscale.blog / indium.tech); no primary study identified | see §9 |
| 10 | 2606.01435 freshness +28 points | BLOCKED — do not cite until re-run | Post-cutoff paper; existence and the ~+28-point delta could not be confirmed during the outage | https://arxiv.org/abs/2606.01435 (unconfirmed) |
| 11 | 2606.31498 protocol governance gaps | BLOCKED — do not cite until re-run | Post-cutoff paper; existence and the MCP/A2A/ACP gap list could not be confirmed during the outage | https://arxiv.org/abs/2606.31498 (unconfirmed) |
| 12 | CAID +25.6% PaperBench | BLOCKED — do not cite until re-run | Post-cutoff paper; +25.6% / +14.7% figures and Geng & Neubig affiliation could not be confirmed during the outage | https://arxiv.org/abs/2603.21489 (unconfirmed) |

## Per-claim detail

### 1. Spark to Fire (arXiv:2603.04474) — VERIFIED

- **Claim as cited:** genealogy-graph message-layer governance prevents final infection in ≥89% of runs.
- **Primary abstract (live fetch, 2026-07-29):** "Experiments show that this approach prevents final infection in at least 89% of runs across operating modes and significantly mitigates the cascading spread of minor errors." Defense described as "a genealogy-graph-based governance layer, implemented as a message-layer plugin."
- **Conditions and baseline (from the repo's full-paper review, `research/papers/spark-to-fire/spark-to-fire.review.md`, status "reviewed-in-full (15 pp)", 2026-06-30):** the ≥89% spans the defense's three operating modes — Speed ~0.89 BICR (cost-aware), Balanced 0.93, Strict 0.94. Baselines (§VII): no-defense containment ≈2.2%; `no_blocking` ablation (detection without rollback) ≈3.1%. So the honest one-liner is: *governance with block+rollback contains 89–94% of runs depending on operating mode, vs ~2–3% without blocking* — detection alone does nothing; rollback/isolation is the active ingredient.
- **Title/authors:** "From Spark to Fire: Modeling and Mitigating Error Cascades in LLM-Based Multi-Agent Collaboration," Xie, Zhu, Zhang, Zhu, Ye, Qi, Chen, Zhou (City Univ. of Macau + Minzu Univ.), v2 May 2026.

### 2. SREGym (arXiv:2605.07161) — PARTIALLY VERIFIED

- **Claim as cited:** frontier agents at 38.9–72.6% diagnosis success on realistic SRE faults.
- **Primary abstract (live fetch, 2026-07-29):** paper exists — "SREGym: A Live Benchmark for AI SRE Agents with High-Fidelity Failure Scenarios" (Clark, Su, Pial, Tian, Gniedziejko, Jacobsen, Chen, Xu). Abstract confirms "SREGym currently includes 90 realistic, challenging SRE problems" and that agents' "capabilities varies significantly in addressing different kinds of failures, with up to 40% differences in end-to-end results."
- **Not confirmed:** the 38.9–72.6% diagnosis range, the 57.3–78.5% mitigation range, and which models — these are full-text results-table numbers; the full-text fetch was blocked by the outage.
- **Cite as (safe today):** the abstract-level facts above. Keep the specific ranges flagged "abstract-unverified" in the spec until the results table is checked (action item below).

### 3. MAST (arXiv:2503.13657) — VERIFIED; affiliation note corrected

- **Primary abstract (live fetch, 2026-07-29):** "Why Do Multi-Agent LLM Systems Fail?" — dataset of "1600+ annotated traces collected across 7 popular MAS frameworks"; taxonomy of "14 unique modes, clustered into 3 categories: (i) system design issues, (ii) inter-agent misalignment, and (iii) task verification"; MAS "performance gains on popular benchmarks are often minimal."
- **Affiliation question resolved:** authors are Cemri, Pan, Yang, Agrawal, Chopra, Tiwari, Keutzer, Parameswaran, Klein, Ramchandran, Zaharia, Gonzalez, Stoica — the UC Berkeley group. **Graham Neubig is not a MAST author at all**; the Neubig question belongs to CAID (item 12), where Neubig's public affiliation is CMU, and the "Anthropic" attribution in one round-1 summary remains unsubstantiated. No Anthropic affiliation on MAST.

### 4. Anthropic ~15× token cost — VERIFIED (pre-cutoff primary; wording should be spot-checked live)

- **Primary source:** Anthropic engineering post "How we built our multi-agent research system" (June 2025) — this IS the primary, and it predates the model knowledge cutoff (Jan 2026), so its content is known even though the live fetch was blocked.
- **Figures from the post:** agents typically use about **4× more tokens** than chat interactions, and **multi-agent systems use about 15× more tokens than chats**; in their BrowseComp analysis, **token usage by itself explains about 80% of the performance variance**. The post's framing: multi-agent architectures work largely because they "spend enough tokens" on the problem — so the economics only justify them for high-value, parallelizable tasks.
- **Note:** the corpus's "~15× the token cost of a chat interaction" is the correct reading (15× vs *chat*, not vs a single agent — a single agent is already ~4×). Spot-check exact sentence wording on the live page before quoting verbatim in external docs.

### 5. arXiv:2602.00496 (junior over-trust) — CORRECTED (nuance)

- **Claim as cited:** "juniors accept agent output readily, sometimes without comprehension."
- **Primary (live fetch, 2026-07-29):** "From Junior to Senior: Allocating Agency and Navigating Professional Growth in Agentic AI-Mediated Software Engineering." Three-phase mixed-methods: ACTA + Delphi with 5 senior engineers; an AI-assisted debugging task with 10 juniors; blind reviews of junior prompt histories by 5 additional seniors.
- **What the abstract actually says:** "novices struggle between over-reliance and cautious avoidance" — juniors mis-calibrate trust in BOTH directions, not uniformly over-trust.
- **Correction for the spec:** cite as "juniors mis-calibrate trust (oscillating between over-reliance and avoidance); seniors retain interrogative oversight." Note the tiny N (10 juniors, 10 seniors) — qualitative signal, not a quantified acceptance rate. The design consequence for the garden is unchanged (evidence-rich approval cards, default-deny on irreversible actions) but the *stated rationale* should be "trust mis-calibration," which also covers the avoidance failure mode (a junior who rubber-stamps NOTHING is also a release risk).

### 6. METR RCT — VERIFIED (pre-cutoff primary; live re-check blocked)

- **Primary source:** METR, "Measuring the Impact of Early-2025 AI on Experienced Open-Source Developer Productivity" (blog, 2025-07-10) — pre-cutoff, content known.
- **Figures:** RCT with 16 experienced open-source developers, 246 tasks in mature repos they knew well. With AI allowed, tasks took **19% longer**. Before the study, developers forecast AI would speed them up ~24%; **after experiencing the slowdown, they still estimated AI had sped them up ~20%**. So "19% slower while believing they were 20% faster" is accurate.
- **Context caveat (from round 1, stands):** METR has since labeled the result historical and changed its experiment design (metr.org/blog/2026-02-24-uplift-update/). Cite it for the *perception gap* (self-reports are unreliable; measure), not as a current-tools productivity estimate.

### 7. "Anthropic 47% debugging-skill drop" — UNVERIFIABLE — do not cite

- Round 1 (`notes/research-practitioner-lessons.md` §4, §"Unverified") already found this only in secondary blogs, never with a link to any Anthropic publication. Model knowledge through Jan 2026 likewise contains **no Anthropic study reporting a 47% debugging-skill drop**; Anthropic's related public work (economic index, skill-formation commentary) contains no such figure.
- The live search sweep to catch a post-Jan-2026 primary was blocked by the outage, so this is "no primary found by two independent passes," not a proof of nonexistence — but the operational verdict stands: **do not cite**. The adjacent claim that IS citable: the 52-junior RCT with a 17-point comprehension/debugging quiz gap (via Osmani's summary; itself worth a primary-source chase before external use).

### 8. $47,000 LangChain loop + InfoQ-verified incidents

- **$47k loop — UNVERIFIABLE, do not cite.** Round 1 found it only in two dev.to posts with no named company, no invoice, no independent coverage. Nothing in model knowledge corroborates it. Keep the corpus's framing: illustrative anecdote only — and note the *design* lesson (any per-task cap under $200 kills it) survives without the anecdote, since the InfoQ incidents make the same point.
- **InfoQ pair — round-1 verification stands as described.** `notes/research-emergent-3.md` §5 records them from the InfoQ article (July 2026, infoq.com/news/2026/07/ai-agents-billing-guardrails/): (a) 3-person AWS agency, normal bill $10–15/mo, **$14,000 in one day** after attackers extracted static EC2 access keys and burned Bedrock Claude invocations; (b) DN42 hobby-network operator (May 2026) gave an autonomous agent full AWS access; it provisioned five m8g.12xlarge instances, LBs, Lambdas, re-applied its CloudFormation template repeatedly — bill **$6,531.30, negotiated to $1,894**. The outage prevented a fresh fetch of the article; no contrary information encountered. Confidence: high (first-party trade-press reporting, fetched in round 1 three days before this pass).

### 9. ">4h agents without persistence carry ~90% higher total-failure risk" — UNVERIFIABLE — do not cite

- **Origin:** round 1 (`notes/research-arxiv-multiagent-sdlc.md` §7) sourced the durability-practice consensus to three industry blogs (slavadubrov.github.io 2026-05-26; appscale.blog durable-execution piece; indium.tech "7 state persistence strategies") and explicitly tagged this one number "unverified vendor number." It appears in none of the arXiv material surveyed.
- The outage blocked fetching the three blogs to pin down which one carries it and what THEY cite. Regardless of provenance hunt outcome, a vendor-blog risk multiplier with no methodology is not spec-grade: **do not cite the number**. The qualitative consensus (checkpoint + resume + heartbeats for long-running agents) is independently well-supported and is what the spec should lean on.

### 10. arXiv:2606.01435 ("Don't Ask the LLM to Track Freshness") — BLOCKED — do not cite until re-run

- Post-cutoff paper (June 2026); its existence, exact title, and the "~+28 points" delta for deterministic freshness resolution vs LLM adjudication could not be confirmed — every fetch attempt during this pass hit the classifier outage. Round 1 itself flags the 2606-series numbers as "abstract-level only, not deeply verified."
- **Action:** re-run the abstract fetch; confirm the benchmark name and whether +28 is points of accuracy, and on which condition. Until then the spec may keep the *design principle* (resolve freshness deterministically in pipeline code; the supporting evidence base — Zep/Graphiti, FRESCO, temporal-RAG recency priors — is broader than this one paper) but not the +28 number.

### 11. arXiv:2606.31498 (governance gaps in MCP/A2A/ACP) — BLOCKED — do not cite until re-run

- Post-cutoff paper; existence and the exact claimed gap list (granular contextual permissions, constrained delegation chains, distributed audit, budgets/rate limits) could not be confirmed during the outage. Note also the ID's sequence number (31498) is unusually high for a monthly arXiv series — worth confirming the ID itself is transcribed correctly, not just the content.
- **Action:** fetch the abs page; if the ID 404s, search the title "Governance Gaps in Agent Interoperability Protocols" (authors given in round 1 as Kang & Diponegoro). The differentiation argument in `00-research-summary.md` leans on this paper — it should not harden into the spec on an unconfirmed citation.

### 12. arXiv:2603.21489 (CAID) — BLOCKED — do not cite until re-run

- Post-cutoff paper (March 2026, rev. July 2026); the +25.6% PaperBench / +14.7% Commit0 figures, the baseline they're measured against, and the author affiliation could not be confirmed during the outage.
- **Affiliation note:** round 1 flagged that an automated summary attributed the authors to Anthropic while Neubig is publicly at CMU. This check is still owed. (The MAST/Neubig confusion in item 3 is resolved: Neubig has nothing to do with MAST.)
- **Action:** fetch the abs page; confirm figures, whether +25.6% is absolute or relative, the baseline (single synchronous agent? no-isolation delegation?), and affiliations.

## Outstanding re-run list (blocked by 2026-07-29 tool outage)

1. SREGym full text — confirm 38.9–72.6% diagnosis / 57.3–78.5% mitigation and model names.
2. arXiv 2606.01435 — existence + exact +28-point delta + benchmark.
3. arXiv 2606.31498 — existence (check ID) + exact gap list.
4. arXiv 2603.21489 — +25.6%/+14.7% + baseline + affiliations.
5. Spot-check exact 15×/80% sentence wording on the live Anthropic post; exact 19%/20% wording on the live METR post.
6. Fresh fetch of the InfoQ billing-guardrails article to re-confirm the $14k/$6.5k details.
7. Post-Jan-2026 search sweep for any primary behind "Anthropic 47%" (expected result: none) and for the origin of the ">4h/90%" vendor number.
