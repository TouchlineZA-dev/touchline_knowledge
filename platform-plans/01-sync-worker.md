# Touchline sync worker and database plan

**Program** — `/Users/phathusigama/Documents/touchline`, local `main` at `eb28093`.  
**Outcome** — Sportmonks supplies football facts to one scheduled worker. The C# API reads those facts and owns fantasy writes. Flyway in this repo publishes the database changes for both programs.

## Establish what is running

The repo contains two implementations of the football schedule. FastAPI starts APScheduler from `app/main.py`; `netlify/functions/sync-daily.mts` and `sync-fixtures.mts` also schedule ingestion. The Python schedule reconciles the season daily at 02:00 SAST and refreshes a rolling fixture window hourly. Players come from fixture lineups. The Netlify news job also supplies the web app’s RSS cache. Removing all three jobs together would risk losing headlines.

Use the always-on Python worker described in `render.yaml` as the football writer, matching the supplied architecture. Inventory actual deployments, active schedules, database roles, last successful jobs and news consumers first. Compare old and new football output in separate staging data. Hand off the schedule at a recorded checkpoint, with one active writer per environment.

## First delivery — make the shared schema reproducible

1. Compare staging Flyway history, columns, constraints and grants with `db/migrations/V001…V032`. Check for manual changes, frontend proposal tables, duplicate points rows and existing teams. Record row counts and a restorable backup before applying changes. The checkout establishes what is versioned. Staging establishes what is installed.
2. Recover the proposed V033 from the API history immediately before deletion commit `9bac9d2`. Review the nullable, unique UUID `app.users.auth_user_id` and nullable `password_hash`. Existing profiles keep their integer IDs and team relationships. Resolve legacy email conflicts through verified account ownership.
3. Review the API repo’s `docs/proposed-migrations/V034__fantasy_core.sql` against the actual baseline. It depends on the V031 squad constraint, V002 timestamp function and unique points keys. Check existing season IDs, budget and price values, captain indexes, triggers, sequence access and grants. Changing a default affects future inserts. Reconcile existing balances and season assignments explicitly. Use the next available migration versions if staging history already occupies 33 or 34.
4. Rehearse both a clean installation and an upgrade of a populated staging copy. Have API CI consume the released migration files. Keep migration administration separate from `sync_worker_user` and `app_api_user`. V001 contains literal role passwords. Rotate any matching deployed credentials and provision fresh installations with separately managed secrets while preserving applied Flyway checksums.

## Second delivery — make football data usable by fantasy

- Align season selection with C#. Python can fall back to a populated previous season while C# selects an `is_current` PSL season. Define the current competition and season contract, including preseason, bootstrap and rollover. Test a new current season with zero fixtures alongside a populated previous season.
- Load the PSL reference data, fixtures, lineups, events and stats. Preserve internal database IDs for app relationships and Sportmonks IDs for event and lineup joins. Hand the API team representative completed fixtures with starters, bench players and substitutions, plus an upcoming fixture for deadline testing.
- Implement a writer for `sync.gameweek_sync_status`. The model and V012 table exist, but the scheduled code has no writer for those flags. Agree with the API owner how to treat missing events, a match with zero events, delayed stats and postponed fixtures. Publish completeness after the relevant ingestion commits. Reconcile upstream corrections, including removed events, and trigger an explicit rescore when finalized facts change.
- Treat the current hourly cadence as an hourly feed. Set a matchday freshness target against provider quota before showing live updates. Run more frequent refreshes for active matches if required. Track last successful fixture refresh independently of general process health.
- Require real production settings. `app/config.py` currently permits a local SQLite URL and dummy keys. Set the v3 football base path used by the deployed worker. Give readiness a failing status on lost database access. Retain one scheduler process and make job acquisition atomic across manual triggers and scheduled runs.

## Acceptance and handoff

Run the same fixture import twice. Counts, internal IDs and associated events stay stable. A corrected fixture updates the existing record. The API sees the same season, match and players. A complete gameweek becomes ready and an incomplete one remains pending. Rehearse a worker restart, provider failure and recovery. Hand the API owner the migration version, sample fixture IDs, last successful run and gameweek status.

**Local check** — The 11 existing Python tests passed with dummy settings. They cover retry, mapping and alert behavior. The database upgrade and scheduler handoff need staging. Coordinate news retirement separately with [the news plan](05-news-service.md).
