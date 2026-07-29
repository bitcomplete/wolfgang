# Competitor research: Kasava (kasava.dev)

Researched: 2026-07-29, via web search + page fetches. All claims cited; verified facts vs inference marked throughout.

## Name-collision caveat

"Kasava" is an ambiguous name. Confirmed distinct entities that are **not** the subject of this research:

- **kasava.io** — "AI-Powered Fleet Management" (unrelated) — https://www.kasava.io/
- **Kasava** Google Play app (`io.kasava.kasava`) — https://play.google.com/store/apps/details?id=io.kasava.kasava
- **Kasava Lifestyle** (PitchBook profile) and **KaSaVa** (Crunchbase) — appear to be different companies; the Crunchbase/PitchBook results found in search did not match the dev-tools startup, so **no funding data could be verified** for kasava.dev.
- **KASAVA GROUP** (LinkedIn, Sébastien Asteggiano) — unrelated.

This document covers only **kasava.dev**, the SF dev-tools startup (LinkedIn: `kasava-ai`).

## What Kasava is (verified)

- **Current positioning (homepage, July 2026):** "PRDs from your real code, not a blank page." Kasava reads your GitHub repos, Gong call transcripts, and Notion docs, then drafts Product Requirements Documents; you edit and push epics/tickets to Linear or Jira. (https://www.kasava.dev/)
- **Broader/earlier positioning (page titles, docs, LinkedIn):** "The Agentic Platform for Product Engineers" / "the only AI-native platform purpose-built for product engineers" — a unified hub for issues, PRs, and tasks across platforms, semantic code search, AI chat over the codebase, AI-generated digests. (https://www.kasava.dev/blog title tag, https://www.linkedin.com/company/kasava-ai, https://www.kasava.dev/docs)
- **Inference:** the homepage's narrow PRD-first pitch versus the broader "agentic platform" framing elsewhere suggests a recent narrowing/repositioning. Supporting signal: an April 2026 blog post framed as "stripping down" complexity from an initial version (per blog index summary, https://www.kasava.dev/blog). Treat the pivot as inferred, not stated.

### Product shape (verified from site + docs)

- **Product Graph** — unified index over repos, specs, docs, transcripts; cross-referenced queries. (homepage)
- **AI Chat** — Q&A over the codebase with citations to files/PRs/docs; workspace isolation; @mentions of GitHub/Linear/Jira/Notion; custom action templates. (https://www.kasava.dev/features/ai-chat)
- **Research module** — parses Gong calls and competitor websites, clusters recurring asks/bugs. (homepage)
- **Plan generation (PRDs)** — drafts epics/tickets exportable to Linear, Jira, GitHub. (homepage)
- **Commit intelligence** — "Every PR rated for size, risk, and what it actually changed"; automated standup-style summaries. (homepage)
- **Updates/digests** — AI-generated summaries of wins, risks, progress. (https://www.kasava.dev/docs)
- **Bug reports via Chrome extension** — bugs captured with full context. (search snippet of kasava.dev; feature exists but was not prominent on the fetched pages — lightly marketed)
- **Bidirectional sync** with GitHub, Linear, Jira, Asana ("changes you make flow back to the original platform"). (https://www.kasava.dev/docs)
- **Integrations:** GitHub, Linear, Jira, Asana, Gong, Notion, Slack, Sentry, Google Docs, and — notably — **Claude Code**. (homepage; https://www.kasava.dev/integrations)

### How it uses agents (verified + gaps)

- AI is used for retrieval-augmented chat with citations ("every draft cites its sources"), clustering themes from calls/sites, drafting PRDs/tickets, and rating PRs. (homepage, features pages)
- Marketing copy mentions "AI agents that can perform complex multi-step tasks autonomously" (search snippet of docs), but the fetched docs and feature pages contain **no technical detail on agent autonomy, architecture, runtime operations, monitoring, or incident response** — the docs read as marketing/onboarding, not architecture. (https://www.kasava.dev/docs, https://www.kasava.dev/features/ai-chat)
- Company philosophy (blog): "AI is an exoskeleton, not a coworker" — AI as amplifier of a human product engineer, explicitly **not** autonomous-agent-first. (https://www.kasava.dev/blog/ai-as-exoskeleton)

### Target user (verified)

Product managers and "product engineers" on product/engineering teams — people who aggregate context from code, calls, and docs to plan work. Homepage: "PMs are drowning in self-built context systems — a high-effort workaround Kasava can productionize." **Not** operations engineers, SREs, or anyone running apps in production.

## Pricing (verified, https://www.kasava.dev/pricing)

| Tier | Price | Included |
|---|---|---|
| Free | $0 | 300 AI credits/mo, 1 repo, 128 MB, GitHub + Linear only |
| Professional | $29/seat/mo (annual, 17% discount noted) | 1.5K credits/seat/mo, 10 repos, 1 GB, all integrations, commit intelligence; 14-day trial, no card |
| Enterprise | Custom | Unlimited repos/storage, BYO-key, SSO/SAML, on-prem, dedicated AM |

Note: one search snippet said "$15/seat/month" — the fetched pricing page says $29/seat/mo; possibly a stale cache or monthly-vs-annual difference. Treat $29 (annual) as the verified figure. Credit-based metering; they published a transparency post "Transparent Pricing: How We Calculate AI Credits" (Jan 24, 2026, blog index).

## Company & maturity (verified where cited)

- **Founder:** Ben Gregory (HN handle `benbeingbin`), California; Stanford GSB MBA; previously Flexport, Mosaic, Monroe, Coplay. (https://rocketreach.co/ben-gregory-email_125321885 — third-party data, treat as probable not certain)
- **Company:** San Francisco, founded 2025, **2–10 employees**, 28 LinkedIn followers. (https://www.linkedin.com/company/kasava-ai)
- **Funding:** none verifiable (see name-collision caveat). Inference: likely bootstrapped or pre-seed.
- **Engineering:** whole company in one TypeScript monorepo (5,470+ files) — Next.js frontend on Vercel, backend on Cloudflare Workers, Chrome extension, Google Docs add-on; CLAUDE.md files per directory; heavy Claude Code usage; "Everything ships the same way: git push." (https://www.kasava.dev/blog/everything-as-code-monorepo)
- **Maturity as of 2026-07:** live product with free tier, trial, docs, ~10 integrations, and an active blog since ~Sep 2025. Still very early: tiny team, tiny audience, docs.kasava.dev had an **expired TLS certificate** when fetched (2026-07-29) — a concrete operational-polish gap.

## Public reception (verified)

- Blog content marketing works well on HN; the **product** gets little engagement:
  - "AI is not a coworker, it's an exoskeleton" — **522 points, 576 comments** (2026-02-19), but discussion was about the metaphor/AI-jobs debate, largely skeptical; one commenter asked "Can you highlight what you've managed to do with it?" with no substantive reply visible. (https://news.ycombinator.com/item?id=47078324)
  - "Everything as code: How we manage our company in one monorepo" — **227 points** (2025-12-30). (https://news.ycombinator.com/item?id=46437381)
  - Three other submissions ≤4 points. (hn.algolia.com search "kasava")
- No Product Hunt launch, third-party reviews, or named customers found in searches. **No public traction evidence beyond blog virality.**

## Relevance to the software garden

### Where Kasava overlaps

- **Knowledge base grounded in real artifacts:** Product Graph = unified, citation-backed index over code/docs/conversations. Same instinct as the garden's knowledge/navigator layer (and Wolfgang itself). Their "every draft cites its sources" discipline mirrors our grounding rules.
- **Structured planning output:** PRDs → epics → tickets pushed to Linear/Jira is close to Wolfgang's specs-and-work-breakdown role.
- **Digests/updates:** AI summaries of wins/risks/progress overlap with garden reporting ideas (cf. bc-generate-sitrep).
- **Claude Code as first-class integration** and CLAUDE.md-per-directory practice — validates the garden's bet that repos should be structured as agent context.

### Where Kasava does NOT compete

- **Runtime operations is absent.** No monitoring, alerting, incident response, deploys, or on-call features anywhere in the fetched docs/features (Sentry appears only as a data source). The garden's July release — a junior engineer operating 4–5 apps in production — is **uncontested by Kasava**.
- **Different user:** PMs/product engineers doing planning, vs. our junior engineer doing ops. Kasava assumes the user already owns product context; our first user needs guardrails and operational runbooks.
- **Human-amplifier philosophy:** their "exoskeleton" stance means they are deliberately not building autonomous agents that act on systems — the garden's agent-bus/agent-involvement model goes further.

### Lessons / implications for the July release and the junior-engineer user

1. **Citation-grounded answers are table stakes.** Kasava leads with "quotes specific code, commits, and docs with citations." Any garden Q&A surface for the junior engineer should cite runbooks/configs/logs the same way — especially important for a junior user who can't judge hallucinations.
2. **Credit-based pricing transparency** (published formula) is a pattern worth copying if/when the garden is metered.
3. **Small-team feasibility proof:** 2–10 people ship a multi-surface product (web, Workers, Chrome extension, Docs add-on) by going all-in on monorepo + Claude Code + everything-as-code. Directly applicable to how Bit Complete builds the garden itself.
4. **Cautionary tale on positioning:** "agentic platform for product engineers" → narrowed to "PRDs from code" within ~a year (inferred). Broad "platform" pitches don't land; the garden's first release should lead with the concrete job (run these 4–5 apps) not the ecosystem vision.
5. **Content ≠ traction:** 522-point HN posts produced no visible product adoption evidence. Blog virality won't substitute for a working wedge.
6. **Operational polish matters:** an expired cert on their docs domain is exactly the kind of failure the garden's ops tooling should catch for its own apps (cert expiry monitoring is a concrete junior-engineer-friendly feature candidate).

## Verified vs inferred — quick key

- **Verified:** positioning copy, features, integrations, pricing tiers, HN numbers, LinkedIn size/founding, monorepo/stack details, expired docs cert.
- **Third-party / probable:** founder identity (RocketReach).
- **Inferred:** repositioning/pivot, bootstrapped status, "no traction" (absence of evidence), degree of agent autonomy.
- **Unknown:** revenue, customers, funding, team roster, roadmap toward operations.

## Sources

- https://www.kasava.dev/ (homepage)
- https://www.kasava.dev/pricing
- https://www.kasava.dev/docs (docs.kasava.dev had an expired TLS cert on 2026-07-29)
- https://www.kasava.dev/features/ai-chat
- https://www.kasava.dev/blog (index)
- https://www.kasava.dev/blog/everything-as-code-monorepo
- https://www.kasava.dev/blog/ai-as-exoskeleton
- https://www.kasava.dev/integrations
- https://news.ycombinator.com/item?id=47078324 (exoskeleton HN thread)
- https://news.ycombinator.com/item?id=46437381 (monorepo HN thread)
- https://hn.algolia.com/api/v1/search?query=kasava
- https://www.linkedin.com/company/kasava-ai
- https://rocketreach.co/ben-gregory-email_125321885
- Collision references: https://www.kasava.io/, https://play.google.com/store/apps/details?id=io.kasava.kasava
