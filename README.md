# Wolfgang

Wolfgang is a **project-planning and ideation navigator** — a thinking partner for
designing systems. It pressure-tests ideas, weighs trade-offs, records decisions with
their rationale, and turns open-ended discussion into specs and buildable work.

Its flagship design work lives in [`research/`](./research): **Greenwood**, an
event-sourced agent runtime and coordination bus (built on Kafka) whose organizing idea
is to preserve the *genealogy* of agentic interactions so faulty branches can be
monitored and pruned before errors cascade.

## Greenwood at a glance
An event-sourced agent runtime where every action is an immutable event and all state is
a fold of the log. That one property does a lot of work:
- **Resume** — a rescheduled agent is rebuilt by deterministic replay, hitting the LLM's
  content-addressed prompt cache from a fresh host.
- **Genealogy handoff** — an agent hands its successor a *verified slice of the lineage
  graph*, not a transcript, so context is traceable and errors are gated at the boundary.
- **Coldframe (evals)** — human feedback becomes replayable eval cases; failures are
  reproduced, the harness is refined, and the case joins a regression suite.

Start with [`research/decisions.md`](./research/decisions.md) and
[`research/NAMES.md`](./research/NAMES.md), then
[`research/WORK-BREAKDOWN.md`](./research/WORK-BREAKDOWN.md).

## Repository layout
- `research/` — the Greenwood design: decisions log, design topics, work breakdown
- `CLAUDE.md` — navigator instructions
- `config.json` — navigator configuration
- `knowledge/` — the navigator's knowledge base
- `.autonav/` — navigator skills (`ask` / `update`)

## Using the navigator
Wolfgang runs on the Autonav CLI:

```bash
autonav query wolfgang "your question here"
```
