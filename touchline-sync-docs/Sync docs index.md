# Sync repository docs (snapshot)

[Team home](../MASTER.md) · [Football and news sync](../touchline_sync.md) · [Sync Sprints 3–5](../touchline_sync_sprints_3-5.md)

These are copies of every Markdown document in the `touchline-sync` repository. They were taken at commit `cf69e84` on branch `sprint/6-api-handoff-fixes` on 24 Nov 2026. That branch is pushed but not merged.

The repository is the source of truth. If a copy here disagrees with the repository, the repository wins: refresh the copy and update the commit above.

| Document | What it explains |
|---|---|
| [README](README.md) | What the Python worker does, setup, environment variables, running it locally |
| [NETLIFY](NETLIFY.md) | The Netlify sync functions, which are the live production writer |
| [BACKEND_INTEGRATION](BACKEND_INTEGRATION.md) | The read contract for the API: tables, states, goal types, budget, short codes |
| [Runbook](docs/runbook.md) | On-call guide: diagnose, restart, reconcile, rotate keys |
| [Sprint 1 • Sportmonks coverage](docs/sprint1/sportmonks-coverage.md) | Which provider data the PSL feed actually has |
| [Sprint 2 • Postgres handoff](docs/sprint2/postgres-handoff.md) | Roles, permissions and the worker-to-API handoff through committed rows |
| [Sprint 3 • Live freshness](docs/sprint3/live-freshness.md) | The live polling cadence, quota, revisions, soft withdrawal (V040) |
| [Sprint 4 • Late changes](docs/sprint4/late-changes.md) | Event history, reschedules, completeness, late corrections (V041) |
| [Sprint 5 • Operations](docs/sprint5/operations.md) | The single-scheduler lease, health checks, alerts, redaction, restore rehearsal (V049, was V042) |
| [Sprint 6 • API handoff fixes](docs/sprint6/api-handoff.md) | Answers to the API team's 13 points (V049 renumbering, V050, passwords) |
