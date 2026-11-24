# Touchline — Sync Worker

Python service that ingests South African football data (Sportmonks) and SA
football news (GNews) into a Supabase PostgreSQL database for the **Touchline**
fantasy football platform. Owns the data pipeline and the migrations; the
user-facing product is a separate backend API (built by a teammate) + frontend.
random change 

## Where this fits in the bigger picture

Touchline has three moving parts, built independently: **this sync worker**,
a **backend API** (a teammate's project — could be ASP.NET, Node, anything),
and a **frontend**. This repo is only the data-ingestion piece — it has no
concept of users, squads, or points, and it never talks to the frontend.

**The sync worker and the backend do not call each other for data — they share
a database instead.** This repo runs on a schedule, fetches from Sportmonks and
GNews, and writes into `football` and `sync` (raw football data + job/cache
bookkeeping). The backend connects to that *same* Supabase database — as a
different, more restricted DB user — reads what this worker wrote, and builds
the actual fantasy game logic (users, squads, scoring, leaderboards) in its own
`app` schema. The frontend only ever talks to the backend; it has no visibility
into this repo at all.

Why not have the backend call this service's API directly for data? Two
reasons:
- **The database is the natural boundary.** Fixtures/players/events are
  relational data best queried with SQL joins, not re-shaped into a bespoke
  HTTP API and re-fetched on every request.
- **Independent failure and schema ownership.** If this worker is down, the
  backend still serves the *last synced* data straight from Postgres — nothing
  breaks. And each service can only write the schema it owns (enforced by DB
  grants below), so a bug on one side can't corrupt the other's tables.

This service does still expose a small internal HTTP API
(`/api/v1/sync/*`, `/api/v1/news/*`), but only as an **optional control
channel** — e.g. the backend can trigger a manual sync — never as the path
data actually flows through. See [Deployment](#deployment) for the full
picture of how the two services connect.

## Architecture

```
Sportmonks API ─┐
                ├─► Python sync worker (this repo) ─► PostgreSQL (Supabase) ─► Backend API ─► Frontend
GNews API ──────┘        FastAPI + APScheduler          football | sync | app
```

Three schemas, split by owner for least-privilege:

| Schema     | Owner            | Contents                                            |
|------------|------------------|-----------------------------------------------------|
| `football` | sync worker      | competitions, seasons, teams, players, fixtures, events, stats, lineups |
| `sync`     | sync worker      | sync_jobs, sync_logs, gameweek_sync_status, news_cache |
| `app`      | backend API      | users, fantasy_teams, fantasy_squads, gameweeks, points_results, leaderboards, player_prices |

Two DB roles: `sync_worker_user` (read/write `football`+`sync`, `SELECT` on
`app.gameweeks` only) and `app_api_user` (read-only `football`+`sync`, read/write
`app`). This app connects **only** as `sync_worker_user`.

### Four rules

1. **Schema ownership is enforced by DB grants.** The sync worker never writes
   `app` tables it doesn't own; the API never writes `football`/`sync`.
2. **Flyway owns all DDL.** Never `Base.metadata.create_all()`. Models carry
   `server_default`s so SQLAlchemy matches the DB, but generate no schema.
3. **Syncs are idempotent** (`INSERT ... ON CONFLICT DO UPDATE`), so re-running
   converges rather than duplicating.
4. **No scheduled player sync — do not re-add it.** The Sportmonks `teamId`
   filter is ignored on our plan and returns the global player set. Real PSL
   players come from fixture **lineups** (`lineups.player` include). See
   `map_players_from_lineups`.

## Setup

```bash
python3.11 -m venv .venv && source .venv/bin/activate
pip install -r requirements-dev.txt      # or requirements.txt for prod only
cp .env.example .env                      # then fill in real values
python -m scripts.verify_supabase         # confirm DB connectivity
./migrate.sh migrate                      # apply all migrations
uvicorn app.main:app --reload             # run the API
```

Docker:

```bash
docker build -t touchline .
docker run --env-file .env -p 8000:8000 touchline   # secrets injected at runtime
```

## Auth contract (for the backend side)

Every `/api/v1/sync/*` and `/api/v1/news/*` endpoint requires a shared secret
header. `/health` is open.

```
X-Internal-Api-Key: <INTERNAL_API_KEY>     # same value in .env and the backend's config
```

Missing header → `422`; wrong value → `401`.

## Scheduled jobs

| Job                | Schedule                     | Does                                              |
|--------------------|------------------------------|---------------------------------------------------|
| `daily_full_sync`  | 02:00 Africa/Johannesburg    | competitions, seasons, teams + full-season fixture reconciliation |
| `fixture_sync`     | every 60 min                 | fixtures in a rolling window (yesterday → +14d) with events/stats/lineups |

A crash-stuck `running` job is swept to `failed` at startup; a manual trigger
while a same-type job runs returns `409`.

## Common commands

```bash
./migrate.sh info                 # migration status
./migrate.sh validate | repair    # validate checksums / repair after a failed migration
python -m pytest tests/ -v        # test suite
python -m scripts.check_models    # models vs live DB (tables + columns)
python -m scripts.check_mappers   # mapper output vs live API
python -m scripts.check_services  # end-to-end backfill (writes real data)
curl localhost:8000/health
LOG_LEVEL=DEBUG uvicorn app.main:app   # + echo every SQL statement
```

## Adding a migration

1. Create `db/migrations/V0NN__short_description.sql` (double underscore; next
   number in sequence).
2. `./migrate.sh migrate`, then `./migrate.sh info` to confirm.
3. Update the matching SQLAlchemy model(s) and run `check_models`.
4. **Never edit a migration that has already run** — Flyway checksums will break.
   Fix forward with a new migration. If one fails mid-way: fix the SQL, then
   `./migrate.sh repair`, then retry.

Admin credentials for Flyway live in `.env` as `ADMIN_DB_*` parts (Session
pooler, port 5432 — never the 6543 Transaction pooler).

## Sportmonks Starter-plan limitations

- **Entity filters are silently ignored** (`?teamId=`, `?seasonId=`,
  `?leagueId=`) — the API returns everything; we filter in Python.
- **The date-window endpoint works** (`/fixtures/between/{from}/{to}`), which is
  why the hourly sync uses it instead of the whole corpus.
- **PSL coverage gaps**: no assists, minutes played, or player ratings on this
  data. That's a data-coverage limit, not something a plan-tier bump fixes —
  fantasy scoring must be designed around what's actually available (goals,
  cards, clean sheets via events, etc.).

## Deployment

**The sync worker and the consuming backend (whatever stack it's built in)
integrate through the shared Supabase database — not through HTTP.** The worker
writes `football`/`sync`; the backend reads them and writes its own `app`
schema. HTTP between the two is an optional control channel, not the data path.

```
   Sportmonks ─┐
               ├─► sync worker ──writes──►  Supabase: football + sync
   GNews ──────┘   (1 container)                       app  ◄── writes ── backend
                        ▲                    reads ────►            ▲
                        └── optional POST /api/v1/sync/* ───────────┘  ◄── frontend
                            (X-Internal-Api-Key header)
```

### Deploy the sync worker

A single Docker container. Render / Railway / Fly.io all fit; a VM with
`docker run` works too. A ready [`render.yaml`](https://github.com/TouchlineZA-dev/touchline-sync/blob/cf69e84/render.yaml) blueprint is
included (single instance, `/health` check, Frankfurt region, secrets prompted
in-dashboard) — on Render, just New → Blueprint → this repo.

1. **Run as a SINGLE instance** (min = max = 1, no autoscaling) — see Operations
   below for why.
2. **Set env vars in the host's secret store** (never a committed `.env`). Full
   list + comments in [`.env.example`](https://github.com/TouchlineZA-dev/touchline-sync/blob/cf69e84/.env.example); the deploy-critical ones:
   ```
   DATABASE_URL=postgresql://sync_worker_user.<ref>:<pw>@aws-0-<region>.pooler.supabase.com:5432/postgres?sslmode=require
   SPORTMONKS_API_KEY=<prod key>
   SPORTMONKS_BASE_URL=https://api.sportmonks.com/v3/football
   GNEWS_API_KEY=<key>
   INTERNAL_API_KEY=<generated; the SAME value goes into the backend>
   ENVIRONMENT=production           # JSON logs
   ALERT_WEBHOOK_URL=<webhook>      # optional, recommended
   ```
3. **Run migrations first, separately.** `./migrate.sh migrate` runs Flyway (via
   Docker) against Supabase using the `ADMIN_DB_*` creds — run it from your
   machine or CI, **not** inside the app container. Order is always: migrate,
   then deploy/restart the app.
4. **Health check path:** `GET /health` (returns liveness + DB reachability).

### Connect a backend

1. **Database (the main link).** Connect to the same Supabase DB as
   `app_api_user` — reads `football`/`sync`, read/write `app`:
   ```
   Host=aws-0-<region>.pooler.supabase.com;Port=5432;Database=postgres;
   Username=app_api_user.<ref>;Password=<pw>;SSL Mode=Require
   ```
   Honour the join contract in "Data caveats" below (join on
   `players.sportmonks_player_id`, never `players.id`).
2. **Optional HTTP trigger** for an on-demand refresh — send the shared secret:
   ```
   POST https://<sync-worker-host>/api/v1/sync/fixtures
   X-Internal-Api-Key: <INTERNAL_API_KEY>     # identical value both sides
   ```
3. **Gameweek handshake (future):** the backend creates rows in `app.gameweeks`;
   the sync worker writes `sync.gameweek_sync_status` to signal when a
   gameweek's data is complete, so the backend knows it's safe to score.

## Operations

- **Deploy as a SINGLE instance.** The scheduler runs in-process; a second
  replica doubles every sync and can race the concurrency guard. Scale out only
  after moving the scheduler to its own service or adding a DB-level job lock.
- **Deploy order:** `./migrate.sh migrate` first, then ship the app. The app
  never migrates on startup.
- **Failure alerting:** set `ALERT_WEBHOOK_URL` (Discord/Slack webhook) and
  every failed sync job — including crash-stuck jobs swept at startup — posts
  there. Unset = silent; you'd have to query `sync.sync_jobs` yourself.
- **Error tracking:** set `SENTRY_DSN` to enable Sentry (errors only, no
  performance tracing). Unset = disabled.
- **CI:** GitHub Actions (`.github/workflows/ci.yml`) runs the test suite and a
  Docker build on every push/PR to main.

## Data caveats for consumers (the backend)

Read these before building the fantasy core against the synced data. Counts are
from the PSL 2024/25 backfill and drift slightly per sync — treat them as
representative, not exact.

- **Player references are joins, not FKs.** `match_events.sportmonks_player_id`,
  `match_events.sportmonks_related_player_id`, and
  `player_lineups.sportmonks_player_id` are **Sportmonks** ids. Join them to
  `football.players.sportmonks_player_id` — never to `players.id`.
- **Some event player-refs won't resolve to a `players` row** (~11 of ~500).
  This is by design: events can reference players who weren't in a captured
  lineup. When attributing points, treat a missing join as expected and fall
  back to the event's `player_name` rather than dropping the event.
- **~1% of lineup rows have a NULL player id** (Sportmonks data gaps). They
  can't be attributed to a player; skip them for per-player scoring.
- **~45% of players have a placeholder `image_path`** (`.../placeholder.png`).
  PSL photo coverage is incomplete — the frontend needs an avatar fallback.
- **No assists / minutes / ratings** (see Starter-plan limits above) — scoring
  must be built from goals, cards, clean sheets, and appearance (lineup) data.
- **Points are computed on the backend side.** The sync worker only populates
  `football` + `sync`; it never writes `app.points_results` or prices.
