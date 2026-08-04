---
title: "Round 2 citation verification — Gemini voice-chat transcript"
date: 2026-08-03
topic: "Verification of three citations from a Gemini voice-chat transcript before KB ingestion"
status: draft
---

# Transcript citation verification (round 2)

Three citations surfaced in a Gemini voice-chat transcript, verified against
primary sources on 2026-08-03 before entering the knowledge base.

## Verdict table

| # | Citation | Verdict | Primary URL |
|---|----------|---------|-------------|
| 1 | ESAA: Event Sourcing for Autonomous Agents in LLM-Based Software Engineering | **VERIFIED** | https://arxiv.org/abs/2602.23193 |
| 2 | Bilateral Attestation of Cross-Organization Agent Actions (IETF I-D, Steven Mih) | **VERIFIED** | https://datatracker.ietf.org/doc/draft-mih-agent-bilateral-attestation/ |
| 3 | "From Spark to Fire" (arXiv 2603.04474) affiliations | **VERIFIED** (with nuance) | https://arxiv.org/abs/2603.04474 |

## 1. ESAA — VERIFIED

- **Exact title:** "ESAA: Event Sourcing for Autonomous Agents in LLM-Based Software Engineering"
- **Author:** Elzo Brito dos Santos Filho (single author)
- **Venue:** arXiv preprint, arXiv:2602.23193 (cs.AI), submitted 2026-02-26. No journal/conference venue.
- **URLs:** https://arxiv.org/abs/2602.23193 · https://arxiv.org/html/2602.23193v1
- **Claimed contribution — accurate.** The paper's core architecture separates the
  agent's cognitive intention from project state mutation, inspired by the Event
  Sourcing pattern: agents emit only structured intentions in validated JSON; a
  deterministic orchestrator validates, persists events to an append-only log
  (`activity.jsonl`), applies file-writing effects, and projects a verifiable
  materialized view (`roadmap.json`). Validated via two case studies, including a
  50-task / 86-event / 4-concurrent-agent run with heterogeneous LLMs.
- **Note:** Transcript gave no author; the transcript's paraphrase ("event-sourced
  append-only logs separating agent cognitive intent from state mutations") matches
  the abstract. Follow-on papers exist (ESAA-Conversational, arXiv 2606.23752;
  ESAA-Security) — distinct works, not the cited one.

## 2. IETF I-D, Bilateral Attestation — VERIFIED

- **Exact title:** "Bilateral Attestation of Cross-Organization Agent Actions"
- **Draft name:** draft-mih-agent-bilateral-attestation (current version -01, dated 2026-07-19)
- **Author:** Steven Mih (Action State Group, Inc.) — matches transcript claim.
- **Status:** Active Internet-Draft, individual submission; not adopted by any WG,
  no intended RFC status.
- **URLs:** https://datatracker.ietf.org/doc/draft-mih-agent-bilateral-attestation/ ·
  https://datatracker.ietf.org/doc/html/draft-mih-agent-bilateral-attestation-01
- **Mechanism summary — matches the transcript claim.** The requesting organization
  signs a *request attestation* binding it to the action and its material terms; the
  performing organization evaluates the request against deterministic constraints at
  the boundary where the action takes effect and signs an *action attestation*
  recording constraint results and disposition (performed / declined / escalated to
  a human) by reference to the request; each party acknowledges the other's
  attestation. The combined record binds each org to its part, and can be anchored
  to a transparency service so a third party trusting neither org can verify it —
  i.e., the "joint tamper-resistant record" in the transcript.
- **Note:** Transcript said "July 2026" — correct for version -01 (2026-07-19).

## 3. From Spark to Fire — affiliations VERIFIED (with nuance)

- **Exact title:** "From Spark to Fire: Modeling and Mitigating Error Cascades in
  LLM-Based Multi-Agent Collaboration"
- **Authors:** Yizhe Xie, Congcong Zhu, Xinyue Zhang, Tianqing Zhu, Dayong Ye,
  Minfeng Qi, Huajie Chen, Wanlei Zhou — arXiv:2603.04474.
- **URLs:** https://arxiv.org/abs/2603.04474 · https://arxiv.org/html/2603.04474
- **Affiliations (from paper header):** All eight authors are affiliated with the
  Faculty of Data Science, City University of Macau. Two authors — Yizhe Xie and
  Xinyue Zhang — hold a dual affiliation with Minzu University of China.
- **Nuance on the transcript claim:** "City University of Macau and the Minzu
  University of China" is accurate as the set of institutions on the paper, but
  slightly overstated as a blanket claim — Minzu University of China applies only to
  two of the eight authors (including first author Xie); City University of Macau is
  the primary affiliation for all authors.
- **KB update suggestion:** record affiliation as "City University of Macau (all
  authors); Minzu University of China (dual affiliation: Y. Xie, X. Zhang)".
