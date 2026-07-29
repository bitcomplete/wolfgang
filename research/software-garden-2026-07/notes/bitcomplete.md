---
doc_type: research-note
project: Greenwood
topic: Bit Complete — company profile and how the software garden fits its strategy
created: 2026-07-29
status: reference
sources: web (bitcomplete.io, GitHub, search aggregators) + local repos (wolfgang, bc-internal)
retrieval: company context for the software-garden late-July release spec
---

# Bit Complete — company research notes

Research date: 2026-07-29. Purpose: characterize the company building the "software
garden" so the late-July release spec is grounded in what this company actually is,
how it operates, and who the first user will be (a junior engineer running runtime
operations for 4–5 apps).

## 1. What Bit Complete is (verified, public)

**Bit Complete Inc.** is a small, bootstrapped, profitable **software development and
consulting firm** founded in **April/May 2020** by **Dylan Trotter** and **Matt
Schweitz**, headquartered in **Toronto** and operating as an **all-remote team across
Canada (Ontario, BC) and Brazil**.

- Homepage positioning: building **AI-enabled applications**, **accelerating product
  development**, and **modernizing infrastructure and operations**; "steered by
  engineering leaders from Google, YouTube, and Slack"; serves "fast-growing startups
  and Fortune 500 companies." Case-study clients named on the site: Snap, OpenSea,
  Parsley Health, Octave, HighNote, Instacart, plus testimonials from HighNote,
  Researchable, and Bennu CEOs. (https://bitcomplete.io)
- Founding story: Trotter's May 2020 post "New beginnings" — after a decade in Bay
  Area tech he moved to Toronto and built the firm around his expertise; focus on
  "designing and building software and organizational processes."
  (https://www.bitcomplete.io/blog/new-beginnings/)
- Careers page (verified on-site): bootstrapped and profitable; all-remote Canada +
  Brazil; profit-sharing for all employees; optional Toronto co-working space;
  culture themes "communication is non-negotiable" and "Giving a Sh!t as a Service";
  4.9 Glassdoor rating claimed. Open roles as of 2026-07: Intermediate Frontend,
  Senior Backend, Senior I Frontend, Senior I & II **Site Reliability Engineer**
  (Canada); Intermediate/Senior I Frontend (Brazil).
  (https://www.bitcomplete.io/careers/)
- Team page names only the two founding partners: **Dylan Trotter** (YouTube,
  Thumbtack; "software as a socio-technical problem") and **Matt Schweitz** (YouTube,
  Slack). (https://www.bitcomplete.io/team/)

### Founders' technical pedigree (verified)
- Dylan Trotter is publicly known as the creator of **Grumpy**, Google/YouTube's
  Python-to-Go transcompiler (2017), and redesigned YouTube's Python application
  server; later Staff SWE at Thumbtack (PHP monolith → GraphQL/SOA migration).
  (https://opensource.googleblog.com/2017/01/grumpy-go-running-python.html,
  https://talkpython.fm/episodes/show/95/grumpy-running-python-on-go,
  https://github.com/trotterdylan)
- This matters for the garden: the founders are systems/runtime people, not just an
  agency brand — a Kafka-based agent runtime is squarely in their historical
  wheelhouse.

## 2. Team scale (verified, with ambiguity)

Third-party aggregators disagree and none is authoritative:
- ZoomInfo: **1–10 employees**, revenue under $5M
  (https://www.zoominfo.com/c/bit-complete-inc/546726790)
- LinkedIn company page: **11–50 employees** (https://ca.linkedin.com/company/bitcomplete)
- Apollo/RocketReach/SignalHire echo similar small-company figures.

**Best estimate: low-teens headcount** — the careers page hiring across 7 roles in
two countries plus profit-sharing language is consistent with a ~10–25 person firm.
This is inference, not a verified number. Practical implication: **no dedicated
platform team of any size stands behind the garden** — whatever ships in July must be
operable by the people who built it plus one junior engineer.

## 3. Products, labs, and public artifacts (verified)

**Bit Complete Labs** (https://www.bitcomplete.io/labs/) is the public experimental
arm — "testing ground for interesting or fun concepts":
- **Viewport Tester** (Oct 2024) — browser-based multi-device visual testing.
- **Promptd** (Sep 2024) — collaboration platform for managing LLM prompts.
- **plz.review** (May 2022) — code-review workflow tool (public repo `plz-cli`).
- GitHub code-review enhancement tooling (Jul 2022).

**GitHub org** (https://github.com/bitcomplete): 23 public repos, Canada, contact
hello@bitcomplete.io. Notable: `sqltestutil` (Go), `gcp-runbatch`, `plz-cli`,
`pin-cli` ("Command-line client for pin — HTML sharing service"),
`bc-mdx-components` ("MDX components for **agent-generated documents**"),
`bc-github-actions`. Go and TypeScript dominate.

The Labs pattern is relevant precedent: Bit Complete routinely productizes internal
developer-experience experiments under its own brand. The software garden is the
largest instance of that pattern to date.

## 4. Internal platform (local evidence, not public)

`bitcomplete.dev` publicly 302-redirects to bitcomplete.io (verified 2026-07-29);
it is the **internal tools domain**. From local repos and this workspace's tooling
(paths: `/Users/terra/Developer/bc-internal`, wolfgang plugin skills):

- **bc-prod** — a shared production Kubernetes cluster (Tailscale-gated kubectl,
  Cloudflare ingress, PostgreSQL/Redis/MinIO provisioning via `bc-prod.yaml`,
  sealed secrets, Grafana, scheduled backups).
- **kploy** — an in-house GitOps deployment system (GitHub App + in-cluster
  operator, Kustomize manifests, tracked images, preview environments). Detailed in
  `/Users/terra/Developer/bc-internal/input/kploy-overview-swot.md`; active
  workstreams in `/Users/terra/Developer/bc-internal/open/` (7 open kploy
  workstreams: preview environments, race conditions, raw-YAML support, etc.).
- **pin.bitcomplete.dev** — internal HTML/document sharing service (agent-shareable
  sessions, plans, insights).
- **bithub.bitcomplete.dev** — internal ops hub (e.g. expense filing).
- **Autonav navigators** (wolfgang, bc-internal, mahdi) — knowledge-base agents used
  for planning and project management.

Inference (well-supported): Bit Complete **runs its client and internal apps on its
own platform** and dogfoods agent workflows throughout. The two open SRE roles
suggest operations of this platform is a real, growing cost center — which is the
economic slot the software garden's "junior engineer operates 4–5 apps" release
targets.

## 5. The software garden in company strategy

From local context (`/Users/terra/Developer/wolfgang/knowledge/README.md`,
`ARCHITECTURE.md`, auto-memory `agent-bus-project.md`):

- "The software garden" is Bit Complete's internal family of agent-infrastructure
  prototypes — **Chattermax** (agent bus), **Chalet** (context refinement),
  **Chizu** (agent knowledge), **Gardener** — deliberately backburnered 2026-06-10
  and mined as prior art.
- The current design, **Greenwood** (this repo, "wolfgang" navigator), is a fresh
  event-sourced agent runtime + coordination bus on Kafka (Annals log, Rootlines
  lineage DAG, Grieve governance, Graft harness adapters), grounded in the *Spark to
  Fire* lineage-governance paper (arXiv 2603.04474). Design is largely settled
  (P0, D1–D8); work breakdown feeds Linear tickets (Linear untouched pending
  authorization).
- None of this is public: no search hit connects Bit Complete to "software garden,"
  Greenwood, Chattermax, etc. The garden is **pre-announcement internal IP**.

**Strategic fit:** Bit Complete's public identity is "consultancy that builds
AI-enabled apps and modernizes infrastructure"; its history (Labs, kploy, bc-prod)
is turning internal leverage into products. The garden is the convergence: an
agent-operations layer over the platform it already runs client apps on. A credible
late-July release is best read as an **internal-productivity milestone with
productization optionality** — the same trajectory as kploy/plz.review — not a
public product launch. It de-risks the consultancy's ops load (junior engineer + 
agents replacing senior on-call attention across 4–5 apps) and, if it works,
becomes the next Labs-style offering with real production miles behind it.

## 6. Implications for the late-July release and its first user

1. **The first user profile is company-realistic.** A junior engineer operating 4–5
   apps matches the bc-prod reality: a handful of client/internal apps, one shared
   cluster, kploy deploys, sealed secrets, Grafana. The release must sit *on top of*
   this exact stack, not a hypothetical one.
2. **Small-company operability bar.** With ~low-teens staff and no platform team,
   the release cannot demand specialist care: the garden's own runtime (Kafka,
   operators, projections) must be at most one notch harder to run than kploy is
   today, or it consumes the SRE capacity it is meant to free.
3. **Junior-first governance is load-bearing, not decorative.** The Grieve screen /
   confirmed-projection design is exactly what lets a junior safely delegate to
   agents; the release spec should treat message screening and rollback as
   MVP-critical, since the human backstop is by definition inexperienced.
4. **Consultancy economics favor boring reliability over demo polish.** Bootstrapped
   and profitable means the release is judged by hours saved on real client-app
   operations, not by launch optics. Prioritize resume-after-failure and safe
   deploy/rollback flows over breadth of agent capabilities.
5. **Reuse the existing internal surface.** pin (sharing), bithub (ops hub), Autonav
   navigators, and kploy's GitHub-native flow are the interfaces the junior engineer
   already touches; the release should surface agent activity through them rather
   than inventing new UIs.
6. **Public silence is strategic room.** Nothing is announced, so the July date is a
   self-imposed internal milestone — scope can trade against the date without
   external cost, but the *first user's* trust cannot.

## 7. Name collisions and verification caveats

- **Do not confuse** Bit Complete with: **bit.dev / teambit** ("AI-powered
  development workspaces," github.com/teambit/bit), **Bitloops** (agent-context
  tooling, bitloops.com), Bitstrips, or Bitport — all surfaced in searches and all
  unrelated.
- "Software garden" searches return **Google's Agent Garden** — unrelated; Bit
  Complete's garden has zero public footprint.
- Employee counts (1–10 vs 11–50) and revenue (<$5M) come from scraped aggregators;
  treat as order-of-magnitude only.
- Client list and testimonials are self-reported on bitcomplete.io; not
  independently verified.
- Wellfound careers page returned HTTP 403; Crunchbase/LinkedIn not directly
  fetched — figures cited via search snippets.

## Sources

Web:
- https://bitcomplete.io (homepage)
- https://www.bitcomplete.io/labs/
- https://www.bitcomplete.io/team/
- https://www.bitcomplete.io/careers/
- https://www.bitcomplete.io/blog/new-beginnings/
- https://github.com/bitcomplete
- https://ca.linkedin.com/company/bitcomplete
- https://www.zoominfo.com/c/bit-complete-inc/546726790
- https://www.crunchbase.com/organization/bit-complete
- https://www.apollo.io/companies/Bit-Complete/5ed9e3d873016200014cfaeb
- https://opensource.googleblog.com/2017/01/grumpy-go-running-python.html
- https://talkpython.fm/episodes/show/95/grumpy-running-python-on-go
- https://www.theregister.com/2017/01/05/googles_grumpy_makes_python_go/

Local:
- /Users/terra/Developer/wolfgang/knowledge/README.md, ARCHITECTURE.md
- /Users/terra/Developer/bc-internal/README.md, input/kploy-overview-swot.md, open/
- ~/.claude-work/.../memory/agent-bus-project.md (auto-memory)
