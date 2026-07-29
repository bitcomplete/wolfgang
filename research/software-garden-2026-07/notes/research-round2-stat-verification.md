---
title: "Round-2 verification: headline statistics against primary sources"
date: 2026-07-29
topic: primary-source verification of 12 statistics cited in software-garden spec discussions
status: draft
---

# Round-2 stat verification (primary sources)

**Method:** each claim was checked against its primary source on 2026-07-29 (arXiv abstract/full-text fetch, first-party post, or the repo's own full-paper review). A multi-hour tool outage interrupted the pass mid-way; the four checks it blocked (items 2 full-text, 10, 11, 12) were completed by live fetch after recovery the same day. Two verdicts (4, 6) rest on model knowledge of pre-2026 primary sources rather than a live fetch — flagged inline; exact-wording spot-checks listed at the end.

Verdicts: **VERIFIED** (exact figure quoted from primary), **CORRECTED** (real figure/claim differs), **UNVERIFIABLE** (no primary source exists — do not cite).

## Verdict table

| # | Claim | Verdict | Exact primary figure | Source |
|---|-------|---------|---------------------|--------|
| 1 | Spark-to-Fire ≥89% cascade prevention | VERIFIED (live fetch + local full-paper review) | "prevents final infection in at least 89% of runs across operating modes" — Speed ~0.89 / Balanced 0.93 / Strict 0.94 BICR vs ≈2.2% no-defense, ≈3.1% detection-only | https://arxiv.org/abs/2603.04474 |
| 2 | SREGym 38.9–72.6% diagnosis | VERIFIED (full text, Table 3) | "diagnosis success rates ranging from 38.9% to 72.6%"; mitigation 57.3–78.5%. Agents: Stratus (Sonnet-4.6, Kimi K2.5), Claude Code (Sonnet-4.6), Codex (GPT-5.4) | https://arxiv.org/abs/2605.07161 |
| 3 | MAST 1,600+ traces; MAS gains minimal | VERIFIED (live fetch); affiliation note CORRECTED | "1600+ annotated traces collected across 7 popular MAS frameworks"; gains "often minimal"; 14 modes / 3 categories. Neubig is NOT a MAST author — it's the UC Berkeley group (Cemri … Stoica) | https://arxiv.org/abs/2503.13657 |
| 4 | Anthropic ~15× token cost | VERIFIED (pre-cutoff primary; live re-check blocked) | Post says agents use ~4× more tokens than chat, and "multi-agent systems use about 15× more tokens than chats"; token usage explains ~80% of performance variance on BrowseComp | https://www.anthropic.com/engineering/built-multi-agent-research-system |
| 5 | 2602.00496 junior over-trust | CORRECTED (live fetch) | Abstract: "novices struggle between over-reliance and cautious avoidance" — mis-calibration in both directions, small-N qualitative (10 juniors, 10 seniors) | https://arxiv.org/abs/2602.00496 |
| 6 | METR 19% slower / believed 20% faster | VERIFIED (pre-cutoff primary; live re-check blocked) | 16 experienced OSS devs, 246 tasks: with AI allowed, completion took 19% longer; post-study, participants estimated AI had sped them up ~20% (pre-study forecast ~24%) | https://metr.org/blog/2025-07-10-early-2025-ai-experienced-os-dev-study/ |
| 7 | Anthropic 47% debugging-skill drop | UNVERIFIABLE — do not cite | No primary Anthropic source found in round 1 (secondary blogs only) and none exists in model knowledge through Jan 2026; live confirmation sweep blocked | none found |
| 8 | $47k LangChain loop; InfoQ $14k / $6.5k incidents | $47k: UNVERIFIABLE — do not cite. InfoQ pair: VERIFIED in round 1; re-check blocked | $47k appears only in two dev.to posts, no named company. InfoQ (July 2026): $14,000/day leaked-key Bedrock incident; DN42 autonomous-provisioning $6,531.30 (AWS negotiated to $1,894) | https://www.infoq.com/news/2026/07/ai-agents-billing-guardrails/ |
| 9 | >4h agents ~90% higher total-failure risk | UNVERIFIABLE — do not cite (origin check blocked) | Round-1 already flags it as an unverified vendor number from industry-blog-grade sources (slavadubrov.github.io / appscale.blog / indium.tech); no primary study identified | see §9 |
| 10 | 2606.01435 freshness +28 points | CORRECTED (live fetch) | +28 points is the MULTI-HOP number vs HippoRAG-v2 on FactConsolidation; single-hop delta is +10.8 points on FC-SH (gpt-4o-mini) via candidate-extraction + max(serial) | https://arxiv.org/abs/2606.01435 |
| 11 | 2606.31498 protocol governance gaps | CORRECTED — major | Paper exists (ID valid) but its six-dimension taxonomy is membership, deliberation, voting, dissent preservation, human escalation, audit/replay. Abstract does NOT claim protocols can't express budgets or delegation constraints | https://arxiv.org/abs/2606.31498 |
| 12 | CAID +25.6% PaperBench | VERIFIED (live fetch) | "25.6% absolute improvement over single-agent baselines" on PaperBench; +14.7% on Commit0. Geng & Neubig; affiliations not shown on abs page (Neubig publicly CMU; no Anthropic evidence) | https://arxiv.org/abs/2603.21489 |

## Per-claim detail

### 1. Spark to Fire (arXiv:2603.04474) — VERIFIED

- **Claim as cited:** genealogy-graph message-layer governance prevents final infection in ≥89% of runs.
- **Primary abstract (live fetch, 2026-07-29):** "Experiments show that this approach prevents final infection in at least 89% of runs across operating modes and significantly mitigates the cascading spread of minor errors." Defense described as "a genealogy-graph-based governance layer, implemented as a message-layer plugin."
- **Conditions and baseline (from the repo's full-paper review, `research/papers/spark-to-fire/spark-to-fire.review.md`, status "reviewed-in-full (15 pp)", 2026-06-30):** the ≥89% spans the defense's three operating modes — Speed ~0.89 BICR (cost-aware), Balanced 0.93, Strict 0.94. Baselines (§VII): no-defense containment ≈2.2%; `no_blocking` ablation (detection without rollback) ≈3.1%. So the honest one-liner is: *governance with block+rollback contains 89–94% of runs depending on operating mode, vs ~2–3% without blocking* — detection alone does nothing; rollback/isolation is the active ingredient.
- **Title/authors:** "From Spark to Fire: Modeling and Mitigating Error Cascades in LLM-Based Multi-Agent Collaboration," Xie, Zhu, Zhang, Zhu, Ye, Qi, Chen, Zhou (City Univ. of Macau + Minzu Univ.), v2 May 2026.

### 2. SREGym (arXiv:2605.07161) — VERIFIED (full text)

- **Claim as cited:** frontier agents at 38.9–72.6% diagnosis success on realistic SRE faults.
- **Primary (abstract + full-text Table 3, fetched 2026-07-29):** "SREGym: A Live Benchmark for AI SRE Agents with High-Fidelity Failure Scenarios" (Clark, Su, Pial, Tian, Gniedziejko, Jacobsen, Chen, Xu). Abstract: "SREGym currently includes 90 realistic, challenging SRE problems"; capability "varies significantly in addressing different kinds of failures, with up to 40% differences in end-to-end results." Results (Table 3): "diagnosis success rates ranging from 38.9% to 72.6%" and "mitigation success rates ranging from 57.3% to 78.5%."
- **Task definition / who was tested:** three agent scaffolds x model backends — Stratus (SRE-specific agent; Claude Sonnet-4.6 and Kimi K2.5), Claude Code (Sonnet-4.6), Codex (GPT-5.4). Diagnosis and mitigation are scored as separate task stages. Claude Code had the highest end-to-end success; Stratus + Sonnet-4.6 the highest mitigation rate.
- **Cite as:** "on SREGym's 90 realistic SRE problems, frontier agent/model pairs diagnose at 38.9–72.6% and mitigate at 57.3–78.5% (Table 3)" — the mitigate-better-than-diagnose reading, and the approval-gates argument built on it, both hold.

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

### 10. arXiv:2606.01435 ("Don't Ask the LLM to Track Freshness") — CORRECTED (precision)

- **Claim as cited:** deterministic freshness resolution beats LLM adjudication by ~+28 points.
- **Primary (live fetch, 2026-07-29):** paper exists — "Don't Ask the LLM to Track Freshness: A Deterministic Recipe for Memory Conflict Resolution" (Reddy & Challaram, submitted 2026-05-31). Core finding confirmed: "the bottleneck on conflict resolution is assembly (post-retrieval aggregation), not storage."
- **Exact numbers:** "replacing the LLM-judgment answer pipeline with candidate-extraction plus Python max(serial) yields **+10.8 points on FC-SH** (gpt-4o-mini)"; the deterministic approach "**beats HippoRAG-v2 by +28 points**" on the **multi-hop** variant of the FactConsolidation benchmark.
- **Correction for the spec:** "~+28 points" is real but it is the multi-hop-vs-HippoRAG-v2 comparison, not a blanket deterministic-vs-LLM delta; the like-for-like pipeline swap is +10.8 on single-hop. Cite as: "+10.8 points (single-hop) from swapping LLM adjudication for deterministic max(serial); up to +28 points vs HippoRAG-v2 on multi-hop FactConsolidation." The design principle (freshness resolution in deterministic pipeline code) is fully supported.

### 11. arXiv:2606.31498 (governance gaps in MCP/A2A/ACP) — CORRECTED — major

- **Claim as cited (round 1 and `00-research-summary.md`):** the protocols "cannot express" granular contextual permissions, delegation chains with constrained sub-agent authority, distributed audit, computational/financial budgets and rate limits.
- **Primary (live fetch, 2026-07-29):** paper exists and the ID is valid — "Governance Gaps in Agent Interoperability Protocols: What MCP, A2A, and ACP Cannot Express" (Kang & Diponegoro). But the abstract's actual taxonomy is **six governance dimensions: membership, deliberation, voting, dissent preservation, human escalation, and audit/replay** (applied across five protocols). Findings: "voting and dissent preservation are universally absent across all five protocols, deliberation is absent or at most partial"; gaps are classified as *extensible* (fixable via extensions) vs *structural* (needing new architectural layers).
- **The abstract does NOT claim the protocols cannot express budgets or delegation constraints.** The round-1 gap list (budgets, rate limits, constrained delegation, granular permissions) appears to be an automated-reader artifact or conflation with another source — it is not supported by this paper's abstract.
- **Consequence for the spec:** the D8/differentiation argument "bus-level governance because MCP/A2A/ACP can't express budgets" **must not cite this paper for the budget/delegation point**. What it CAN be cited for: audit/replay, human-escalation, and deliberation/dissent gaps in current interop protocols. The budget/delegation gap claim needs a different source (or should stand on the garden's own analysis, labeled as such). Flag `00-research-summary.md` line 35 and `research-arxiv-multiagent-sdlc.md` §4/§8.3 for rewording.

### 12. arXiv:2603.21489 (CAID) — VERIFIED

- **Claim as cited:** +25.6% PaperBench via git worktrees + branch-and-merge.
- **Primary (live fetch, 2026-07-29):** "Effective Strategies for Asynchronous Software Engineering Agents," Jiayi Geng & Graham Neubig. CAID confirmed: "constructs dependency-aware task plans through a central manager, executes subtasks concurrently in isolated workspaces, and consolidates progress via structured integration with executable test-based verification."
- **Numbers:** **25.6% absolute improvement over single-agent baselines on PaperBench** (paper reproduction); **+14.7% on Commit0** (Python library development). Baseline = single-agent.
- **Affiliations:** not shown on the abs page; Neubig is publicly CMU. No evidence for the "Anthropic" attribution in the round-1 automated summary — treat that as an artifact and drop it.

## Remaining spot-checks (low priority; verdicts above do not depend on them)

1. Exact 15×/80% sentence wording on the live Anthropic post, and exact 19%/20% wording on the live METR post (both verdicts rest on pre-cutoff knowledge of the primary; wording should be confirmed before verbatim external quoting).
2. Fresh fetch of the InfoQ billing-guardrails article to re-confirm the $14k/$6.5k incident details (round-1 live verification stands).
3. Post-Jan-2026 search sweep for any primary behind "Anthropic 47%" (expected: none) and for the origin of the ">4h/90%" vendor number (both remain do-not-cite regardless).

## Corpus follow-ups triggered by this pass

- Reword `00-research-summary.md` (line 35, §"Differentiation thesis") and `notes/research-arxiv-multiagent-sdlc.md` §4/§8.3: arXiv 2606.31498 does not support the "cannot express budgets/delegation constraints" claim (item 11).
- Soften junior-over-trust phrasing wherever 2602.00496 is cited (`00-research-summary.md` item 6, `reports/hci-for-agents.md`, `reports/agents-for-planning-slicing.md`): "trust mis-calibration in both directions," small-N qualitative (item 5).
- The "~+28 points" freshness figure should be quoted with its conditions (item 10) in `reports/reducing-cognitive-debt.md`, `reports/hci-for-agents.md`, `reports/agents-for-planning-slicing.md`, `notes/research-emergent-1.md`.
