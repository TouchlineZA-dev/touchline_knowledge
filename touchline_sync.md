# Touchline Sync • importing football and news

[Team home](MASTER.md) · [Database and migrations](DB.md) · [API scoring consumer](touchline_api.md) · [UI consumers](touchline_main.md) · [Sprints 3–5 (unmerged)](touchline_sync_sprints_3-5.md)

**Source • checkout named `touchline`, `main@4c550f7`, 23 November 2026.** This repository owns football ingestion and the shared Flyway migration sequence. Phatu reports sync is hosted on Netlify. There are two executable football pipelines. The Netlify site deploys TypeScript functions. The Docker/Render configuration starts a Python process with its own scheduler. Checking which deployment ran a fixture is the first step in debugging stale data.

## What runs when

| Entry point | Trigger in source | Output and consumers |
|---|---|---|
| Netlify `sync-daily.mts` | 00:00 UTC / 02:00 SAST every day | Competitions, seasons, teams by season, full fixture reconciliation for the selected PSL season. The provider fixture scan is paginated across the corpus, then filtered. Check actual function duration and quota. |
| Netlify `sync-fixtures.mts` | Hourly | From yesterday to 14 days ahead using UTC date strings. Upserts fixtures, players, events, fixture statistics and lineups. Web functions and the platform API read those rows. |
| Netlify `sync-news.mts` | Every three hours starting at 00:00 UTC | Pulls South African football RSS, filters and merges recent stories, upserts `sync.news_cache`. The web headlines function reads `sa_football_news` through its own PostgreSQL connection. |
| Python `daily_full_sync` | 02:00 Africa/Johannesburg | Journals a full job, refreshes reference data and reconciles the current season’s fixtures. Computes per-gameweek completeness. |
| Python `fixture_sync` | Every 60 minutes while the process is running | Journals a windowed job, imports fixture children, commits facts and refreshes completeness. The internal router can trigger the same work manually. |

No scheduled score-finalisation or player-pricing job appears in these paths. Those operations belong to the API. Refreshing a phone screen retrieves the latest committed facts and does not trigger an import.

## Follow a Netlify import from the provider to the database

1. A scheduled handler calls `runReferenceSync` or `runFixtureSync` in `netlify/lib/sync.ts`. The handler times the work and logs counts or failure. Its `config.schedule` export sets the cadence. The Netlify publish directory serves a placeholder.
2. `sportmonks.ts` uses the v3 football base URL and the server’s API key. It pages until there are no more records and includes participants, scores, events, statistics, lineups and lineup players. Network errors and selected 429/5xx responses retry up to five attempts. A 401 indicates key/configuration trouble.
3. Reference rows are inserted or updated by unique provider ID. The worker reads maps from provider competition/season/team IDs to internal IDs. The daily run updates competitions before seasons and teams before fixtures, so foreign keys can resolve.
4. Fixture processing checks the season and competition. `mappers.ts` extracts home/away participants, scores, current state and kickoff. The fixture upsert returns its internal row ID. Each child row stores that ID. Players are discovered from lineup player data, so the market can lack players before lineups have been supplied.
5. Separate upserts persist players, events, stats and lineups. A corrected event with the same provider ID updates an existing row. A response that omits a previously delivered event needs explicit reconciliation. Netlify’s writes are separate SQL statements. A failed invocation can leave a fixture header newer than some children. A repeated import can complete those upserts.
6. The API later queries PostgreSQL directly. The Netlify jobs log their own results and do not implement the Python `sync_jobs` journal or its completeness refresh. Looking only at the API’s last-job field can miss a successful Netlify run.

**Season rollover matters.** Netlify scans PSL seasons ordered by starting date and can select a previous season with fixtures when the newly flagged season is empty. Python selects the latest-starting flagged season even if empty, and skips fixture work if nothing is current. The API selects a flagged season for league 806. If two screens appear to have different seasons, compare the provider season selected in each job with the internal season returned by the API.

## Python worker and internal operations

`app/main.py` configures logging, sweeps stale running jobs at startup, starts APScheduler and shuts it down with the application lifespan. `render.yaml` describes one always-on Docker instance with a readiness probe. Starting the app also starts its schedules. Point local development at isolated data and provider credentials.

`app/services/sync_job_service.py` acquires a transaction-scoped advisory lock per job type on PostgreSQL, checks for an existing running job and creates the job row. A same-type manual/scheduled collision results in a 409 or skipped tick. A full job and a fixture job use different keys. A startup sweep marks running jobs older than three hours failed. Netlify functions do not participate in those locks.

`app/client/sportmonks.py` supplies an asynchronous paginated provider client. Pure mappers translate JSON into rows. Services use provider-ID conflict keys. Python commits fixture facts per fixture, then recomputes readiness after the fixture pass. A failure rolls back the open transaction and records a failed job. Already committed fixtures remain. Inspect `sync.sync_logs` by job ID for the last completed stage.

| Python endpoint | Purpose |
|---|---|
| GET `/health` | Process liveness. Can return 200 with database marked unreachable. |
| GET `/ready` | Database check and latest successful fixture-sync time. DB failure returns 503. A missing or older-than-150-minute fixture sync sets a stale flag even when readiness is 200. |
| POST `/api/v1/sync/full`, POST `/api/v1/sync/fixtures` | Start a background job, returning its ID. Poll status before retrying. |
| GET `/api/v1/sync/jobs/{job_id}` | Read the journal entry. Unknown job returns 404. |
| GET `/api/v1/news/headlines`, `/psl`, `/bafana`, `/search`, `/team/{team_name}`, `/player/{player_name}` under the news prefix | Internal cached news queries backed by GNews, with an optional secondary Node feed in the implementation. |

The internal sync/news routers require `X-Internal-Api-Key`. Missing required header returns 422, a wrong key returns 401. Keep it on services. Mobile consumers use the public read APIs.

## Readiness for fantasy

For each gameweek with a matching internal season, `gameweek_status_service.py` gathers fixtures by SAST date. It requires at least one fixture, every fixture in state 5, 7 or 8, and four hours since each kickoff. The events flag additionally requires at least one event and lineup row **for every fixture**. The stats flag requires a stat row for each. All three flags set `marked_complete`. The writer updates changed rows and keeps the mark sticky if later facts disagree. A genuine eventless match can remain pending. Postponements and corrections need an operator decision.

The API’s current finaliser accepts the mark **or** all fixtures final. These are different tests. Agree which evidence authorises an official score before automating finalisation. Example handoff for a match correction: record fixture ID, affected dates, old/new event, gameweek ID, import completion and rescoring outcome. Verify both team totals and league rank after the rerun.

## Three news paths

- **Web RSS.** Netlify fetches Soccer Laduma, iDiski Times, KickOff and a Google News RSS search. It deduplicates by story link, sorts by published time, caps the list at 50, and stores a JSON array with fetch/expiry timestamps. A single feed failure is logged and the others continue. An all-feed failure can overwrite the last good list with an empty one. Check the per-feed counts.
- **Python query cache.** The internal service can query GNews and cache responses per query. The cache writer references `settings.gnews_cache_ttl_minutes`, absent from the checked-in Settings class even though the example environment lists a TTL. Exercise cache miss and expiry before relying on this route.
- **Expo Node service.** The mobile News tab calls the separate Express/Prisma API in Main. It has categories, team tags and a different article representation. If the team adopts one shared feed, preserve RSS coverage and publisher URLs, agree category identifiers, migrate both clients, then retire unused schedules. [Main and news](touchline_main.md)

## Run and observe

| Runtime | Configuration and check |
|---|---|
| Netlify | Site settings supply `DATABASE_URL` for the worker role, `SPORTMONKS_API_KEY` and `SPORTMONKS_BASE_URL`. The database client uses TLS, one connection per instance and disables prepared statements for transaction pooling. Check function-specific logs, selected season, row counts and runtime. |
| FastAPI | In the sync checkout use Python 3.11, a virtual environment, `pip install -r requirements-dev.txt`, then `python -m pytest tests/`. For an isolated worker use `uvicorn app.main:app --port 8000`. Set database, Sportmonks, GNews and internal-key settings. `ENVIRONMENT=production` rejects dummy keys, a non-Postgres URL and the wrong provider base path. |
| Docker/Render | Dockerfile and blueprint start the worker. The baseline defines one instance to keep the in-process schedule reliable. No migration runs during boot. An optional alert webhook and Sentry configuration support diagnosis. |
| Flyway | Migration credentials connect separately through a session/direct connection. Use the [database runbook](DB.md) for validation, a populated rehearsal and the V036 season seed dependency. |

**Verification on 23 November 2026.** All 29 tests in the Python `tests/` suite passed locally with development defaults. These cover behaviour including retry, season selection, mapping, completeness, production config validation and alerts. A real Supabase upgrade, a scheduled Netlify invocation and a full device-to-API score still need separate verification.

**When a match is missing:** check the scheduled function’s latest completion and quota response, the selected provider season, reference lookup success, window dates and the persisted fixture’s internal season. Then compare the API’s current season and requested date bounds. A long daily scan can fail even when the hourly window is succeeding.

**When the score is present but fantasy is blank:** compare child counts, provider player IDs, the gameweek date range, completeness marks and API minute-code configuration. HTTP health alone cannot establish that a substitution or goal has arrived.

**Proposed writer handover:** retain the reported production path while comparing both implementations on isolated staging data. Repeat a fixture twice, include a correction, an empty new season and a postponed game. Compare IDs, counts and last-success evidence. Choose one football writer per environment, stop its predecessor at a recorded checkpoint and verify the next scheduled run. Keep the RSS job active until the web news reader has another working source.

## File catalogue

This revision contains **128 tracked files**. Combine each directory heading with the filename beneath it. The entries name responsibilities and selected declarations or direct source dependencies. Installed libraries, Git metadata, caches and intermediate builds are local machine state. The private `.env` supplies runtime settings. Share setting names through the example files.

### Repository root

| File | Responsibility and connections |
|---|---|
| `.dockerignore` | Excludes development files and unnecessary inputs from the worker image build. |
| `.env.example` | Lists environment setting names and formats. Supply actual values privately per environment. |
| `.gitignore` | Git exclusions for local settings and generated files. |
| `BACKEND_INTEGRATION.md` | Repository reference for backend integration. Compare its assumptions with the inspected code. |
| `Dockerfile` | Image build and launch instructions for the Python worker. |
| `NETLIFY.md` | Repository reference for netlify. Compare its assumptions with the inspected code. |
| `README.md` | Repository reference for . setup and use. Compare its assumptions with the inspected code. |
| `migrate.sh` | Runs Flyway in Docker against a session/direct admin connection with the canonical migrations mounted. |
| `netlify.toml` | Publishes the placeholder site and bundles the three scheduled TypeScript functions. |
| `package-lock.json` | Pins npm package resolutions for reproducible installs. |
| `package.json` | Declares libraries and runnable scripts for this repository. |
| `pytest.ini` | Sets test discovery and pytest behaviour for the Python worker. |
| `render.yaml` | Deployment blueprint for a single always-on Python instance with database readiness checks. |
| `requirements-dev.txt` | Adds tooling for tests and development on top of the worker dependencies. |
| `requirements.txt` | Python runtime dependencies for HTTP, SQL, scheduling, configuration and monitoring. |
| `test-sync.py` | Manual news-client exercise using the configured SQLAlchemy session. Defines `test`. Uses `news_service.py`, `database.py`. |
| `test.db` | Empty local SQLite placeholder in this revision. PostgreSQL hosts the shared schemas. |
| `tsconfig.json` | TypeScript compile scope, target and resolution settings for this build. |

### `.github/workflows`

| File | Responsibility and connections |
|---|---|
| `ci.yml` | Runs Python tests with dummy settings and builds the Docker image. |

### `app`

| File | Responsibility and connections |
|---|---|
| `__init__.py` | Initialises the app Python package. |
| `config.py` | Loads typed settings and enforces real database/provider configuration in production. Defines `Settings`. Referenced by `sportmonks.py`, `database.py`. |
| `database.py` | Creates the SQLAlchemy engine, session factory and request-scoped database dependency. Defines `Base`, `TimestampMixin`, `get_db`. Uses `config.py`. |
| `dependencies.py` | Validates the internal service header on protected operations. Defines `require_internal_key`. Uses `config.py`. |
| `main.py` | Starts and stops APScheduler, registers API routers and exposes liveness and readiness. Defines `lifespan`, `health`, `ready`. Uses `config.py`, `database.py`, `scheduler.py`. |

### `app/client`

| File | Responsibility and connections |
|---|---|
| `__init__.py` | Initialises the app.client Python package. |
| `sportmonks.py` | Asynchronous provider client implementing pagination, retries and fixture enrichment requests. Defines `SportmonksClient`. Uses `config.py`. |

### `app/jobs`

| File | Responsibility and connections |
|---|---|
| `__init__.py` | Initialises the app.jobs Python package. |
| `scheduler.py` | Orchestrates reference and fixture jobs, journals stages and refreshes gameweek status. Defines `run_full_sync`, `run_fixture_sync`, `daily_full_sync`. Uses `sportmonks.py`, `database.py`, `competition_service.py`. |

### `app/mappers`

| File | Responsibility and connections |
|---|---|
| `__init__.py` | Initialises the app.mappers Python package. |
| `competition_mapper.py` | Translates the Sportmonks competition payload to PostgreSQL fields. Defines `map_competition`. Referenced by `competition_service.py`, `check_mappers.py`. |
| `fixture_mapper.py` | Translates the Sportmonks fixture payload to PostgreSQL fields. Defines `map_fixture`, `map_match_events`, `map_player_match_stats`. Referenced by `fixture_service.py`, `check_mappers.py`. |
| `player_mapper.py` | Translates the Sportmonks player payload to PostgreSQL fields. Defines `map_player`. Referenced by `check_mappers.py`. |
| `season_mapper.py` | Translates the Sportmonks season payload to PostgreSQL fields. Defines `map_season`. Referenced by `season_service.py`, `check_mappers.py`. |
| `team_mapper.py` | Translates the Sportmonks team payload to PostgreSQL fields. Defines `map_team`. Referenced by `team_service.py`, `check_mappers.py`. |

### `app/models`

| File | Responsibility and connections |
|---|---|
| `__init__.py` | Initialises the app.models Python package. Uses `competition.py`, `fixture.py`, `gameweek.py`. |
| `competition.py` | SQLAlchemy mapping of competition rows for worker queries and persistence. Defines `Competition`. Uses `database.py`. |
| `fixture.py` | SQLAlchemy mapping of fixture rows for worker queries and persistence. Defines `Fixture`. Uses `database.py`. |
| `gameweek.py` | SQLAlchemy mapping of gameweek rows for worker queries and persistence. Defines `Gameweek`. Uses `database.py`. |
| `gameweek_sync_status.py` | SQLAlchemy mapping of gameweek sync status rows for worker queries and persistence. Defines `GameweekSyncStatus`. Uses `database.py`. |
| `match_event.py` | SQLAlchemy mapping of match event rows for worker queries and persistence. Defines `MatchEvent`. Uses `database.py`. |
| `player.py` | SQLAlchemy mapping of player rows for worker queries and persistence. Defines `Player`. Uses `database.py`. |
| `player_lineup.py` | SQLAlchemy mapping of player lineup rows for worker queries and persistence. Defines `PlayerLineup`. Uses `database.py`. |
| `player_match_stat.py` | SQLAlchemy mapping of player match stat rows for worker queries and persistence. Defines `PlayerMatchStat`. Uses `database.py`. |
| `season.py` | SQLAlchemy mapping of season rows for worker queries and persistence. Defines `Season`. Uses `database.py`. |
| `sync_job.py` | SQLAlchemy mapping of sync job rows for worker queries and persistence. Defines `SyncJob`. Uses `database.py`. |
| `sync_log.py` | SQLAlchemy mapping of sync log rows for worker queries and persistence. Defines `SyncLog`. Uses `database.py`. |
| `team.py` | SQLAlchemy mapping of team rows for worker queries and persistence. Defines `Team`. Uses `database.py`. |

### `app/routers`

| File | Responsibility and connections |
|---|---|
| `__init__.py` | Initialises the app.routers Python package. |
| `news.py` | Internal authenticated FastAPI news lookup routes for headlines, search, teams and players. Defines `get_headlines`, `get_psl_news`, `get_bafana_news`. Uses `database.py`, `dependencies.py`, `news.py`. |
| `sync.py` | Trigger a full or windowed job and read status by job ID. Defines `trigger_full_sync`, `trigger_fixture_sync`, `get_sync_job_status`. Uses `database.py`, `dependencies.py`, `scheduler.py`. |

### `app/schemas`

| File | Responsibility and connections |
|---|---|
| `__init__.py` | Initialises the app.schemas Python package. |
| `news.py` | Pydantic contracts for news records and API response envelopes. Defines `GNewsSource`, `GNewsArticle`, `GNewsResponse`. Referenced by `news.py`, `news_service.py`. |
| `sync.py` | Pydantic response contracts for job-trigger acknowledgements and job status. Defines `SyncJobTriggered`, `SyncJobStatus`. Referenced by `sync.py`. |

### `app/services`

| File | Responsibility and connections |
|---|---|
| `__init__.py` | Initialises the app.services Python package. |
| `competition_service.py` | Imports competition records and upserts the reference rows needed for fixture ingestion. Defines `sync_competitions`. Uses `sportmonks.py`, `competition_mapper.py`, `competition.py`. |
| `fixture_service.py` | Selects the flagged season and upserts each fixture and its related records as a committed unit. Defines `select_current_season`, `get_current_psl_season_id`, `sync_fixtures`. Uses `sportmonks.py`, `fixture_mapper.py`, `competition.py`. |
| `gameweek_status_service.py` | Computes fixture/event/stat completeness by week and persists sticky completion transitions. Defines `FixtureFacts`, `GameweekFlags`, `evaluate_gameweek`. Uses `fixture.py`, `gameweek.py`, `gameweek_sync_status.py`. |
| `news_service.py` | Queries and caches news results, with GNews and secondary feed integration. Defines `SAFootballNewsClient`. Uses `config.py`, `news.py`. |
| `season_service.py` | Imports season records and upserts the reference rows needed for fixture ingestion. Defines `sync_seasons`. Uses `sportmonks.py`, `season_mapper.py`, `competition.py`. |
| `sync_job_service.py` | Uses a per-type database lock, creates journal rows, sweeps stale work and records outcomes. Defines `start_sync_job`, `try_start_sync_job`, `get_running_job`. Uses `sync_job.py`, `sync_log.py`, `alerts.py`. |
| `team_service.py` | Imports team records and upserts the reference rows needed for fixture ingestion. Defines `sync_teams`. Uses `sportmonks.py`, `team_mapper.py`, `season.py`. |

### `app/utils`

| File | Responsibility and connections |
|---|---|
| `__init__.py` | Initialises the app.utils Python package. |
| `alerts.py` | Optionally sends an operations webhook on sync failure. Defines `send_alert`, `alert_job_failure`. Uses `config.py`. |
| `logging.py` | Configures the worker’s console and production structured logging. Defines `setup_logging`. Referenced by `main.py`. |

### `db/migrations`

| File | Responsibility and connections |
|---|---|
| `V001__schemas_roles_permissions.sql` | Flyway V001. Creates the football, sync and app schemas and service roles. |
| `V002__timestamp_trigger.sql` | Flyway V002. Installs the shared updated-at trigger function. |
| `V003__football_competitions.sql` | Flyway V003. Creates competition identities. |
| `V004__football_seasons.sql` | Flyway V004. Links seasons to competitions. |
| `V005__football_teams.sql` | Flyway V005. Creates real club records. |
| `V006__football_players.sql` | Flyway V006. Creates the player identity catalogue. |
| `V007__football_fixtures.sql` | Flyway V007. Creates fixtures and home/away club references. |
| `V008__football_player_match_stats.sql` | Flyway V008. Creates fixture/side statistics. |
| `V009__app_gameweeks.sql` | Flyway V009. Creates calendar date ranges. |
| `V010__sync_jobs.sql` | Flyway V010. Creates the job journal. |
| `V011__sync_logs.sql` | Flyway V011. Attaches log messages to jobs. |
| `V012__sync_gameweek_status.sql` | Flyway V012. Defines per-week completeness flags. |
| `V013__football_match_events.sql` | Flyway V013. Creates event and provider player references. |
| `V014__app_users.sql` | Flyway V014. Creates manager profiles. |
| `V015__app_fantasy_teams.sql` | Flyway V015. Enforces one fantasy team per user/season. |
| `V016__app_fantasy_squads.sql` | Flyway V016. Persists picks and armband flags. |
| `V017__app_points_results.sql` | Flyway V017. Persists player points and JSON breakdowns. |
| `V018__app_leaderboards.sql` | Flyway V018. Creates leaderboard storage. |
| `V019__final_permissions.sql` | Flyway V019. Applies runtime grants to existing tables and sequences. |
| `V020__indexes.sql` | Flyway V020. Indexes common fixture, points and journal queries. |
| `V021__football_player_lineups.sql` | Flyway V021. Creates fixture lineup rows. |
| `V022__app_player_prices.sql` | Flyway V022. Creates a current price per player/season. |
| `V023__app_player_price_changes.sql` | Flyway V023. Creates price movement history. |
| `V024__fantasy_squads_purchase_price.sql` | Flyway V024. Adds per-pick purchase cost. |
| `V025__fantasy_teams_budget.sql` | Flyway V025. Adds the team balance with an initial 300.0 default. |
| `V026__news_cache.sql` | Flyway V026. Introduces JSON news cache in app. |
| `V027__player_lineups_permissions.sql` | Flyway V027. Adds missing lineup grants. |
| `V028__move_news_cache_and_default_privileges.sql` | Flyway V028. Moves the cache to sync and sets future privileges. |
| `V029__sync_jobs_job_type.sql` | Flyway V029. Backfills job type and indexes job type/status. |
| `V030__rename_sportmonks_player_columns_and_indexes.sql` | Flyway V030. Renames provider player references and adds lookup indexes. |
| `V031__fantasy_integrity_constraints.sql` | Flyway V031. Adds squad, captain and readiness uniqueness plus gameweek season FK. |
| `V032__bigint_sportmonks_ids_and_status_check.sql` | Flyway V032. Widens provider IDs and validates job state. |
| `V033__app_users_supabase_auth_link.sql` | Flyway V033. Links the Auth subject and permits a null legacy hash. |
| `V034__app_price_table_grants.sql` | Flyway V034. Grants price table and sequence access. |
| `V035__fantasy_core.sql` | Flyway V035. Adds fantasy entries, chips, transfers, leagues and valuation controls and revises defaults. |
| `V036__seed_gameweeks_2026_27.sql` | Flyway V036. Seeds 2026/27 gameweeks when the PSL season row exists. |
| `V037__fantasy_rls_policies_for_api.sql` | Flyway V037. Enables RLS and API policies on seven fantasy tables. |
| `V038__player_availability_overrides.sql` | Flyway V038. Adds reasoned availability overrides and their access policy. |

### `db`

| File | Responsibility and connections |
|---|---|
| `supabase_schema.sql` | Reference snapshot explicitly cut at V025. Use the versioned Flyway directory for upgrades. |

### `netlify/functions`

| File | Responsibility and connections |
|---|---|
| `sync-daily.mts` | Daily Netlify reference refresh and full-season fixture reconciliation with run logging. Defines `config`. Uses `sync.ts`. |
| `sync-fixtures.mts` | Hourly Netlify rolling fixture import with counts and failure reporting. Defines `config`. Uses `sync.ts`. |
| `sync-news.mts` | Three-hourly Netlify RSS collection writing the shared headline cache. Defines `config`. Uses `news.ts`. |

### `netlify/lib`

| File | Responsibility and connections |
|---|---|
| `db.ts` | Creates the pooled SQL client, loads provider-to-internal ID maps and selects a season. Defines `sql`, `excludedSet`, `Lookup`. Referenced by `mappers.ts`, `news.ts`. |
| `mappers.ts` | Maps provider reference, fixture, player, statistic, event and lineup payloads to database columns. Defines `mapFixture`, `mapMatchEvents`, `mapPlayerMatchStats`. Uses `db.ts`. |
| `news.ts` | Merges publisher RSS, filters football, deduplicates by URL and replaces the named cache entry. Defines `runNewsSync`. Uses `db.ts`. |
| `sportmonks.ts` | Paginates provider HTTP responses with retry and collects the related fixture data. Defines `FIXTURE_INCLUDES`. Referenced by `sync.ts`. |
| `sync.ts` | Coordinates SQL upserts for the full reconciliation or rolling window using mapped internal IDs. Defines `runFixtureSync`, `runReferenceSync`. Uses `db.ts`, `sportmonks.ts`, `mappers.ts`. |

### `public`

| File | Responsibility and connections |
|---|---|
| `index.html` | Minimal publishable site for the worker’s Netlify deployment. |

### `scripts`

| File | Responsibility and connections |
|---|---|
| `__init__.py` | Initialises the scripts Python package. |
| `check_mappers.py` | Fetches actual provider sample records to inspect their mapped fields. Defines `main`. Uses `sportmonks.py`, `database.py`, `competition_mapper.py`. |
| `check_models.py` | Compares SQLAlchemy models against the configured database tables and columns. Defines `check_models`. Uses `database.py`. |
| `check_services.py` | Exercises real ingestion against the configured provider and database and records a job. Defines `main`. Uses `sportmonks.py`, `database.py`, `competition.py`. |
| `verify_supabase.py` | Diagnoses connection settings and reachability before a migration. Defines `main`. |

### `tests`

| File | Responsibility and connections |
|---|---|
| `__init__.py` | Initialises the tests Python package. |
| `test_config.py` | Regression coverage for config. Uses `config.py`. |

### `tests/client`

| File | Responsibility and connections |
|---|---|
| `__init__.py` | Initialises the tests.client Python package. |
| `test_sportmonks_retry.py` | Regression coverage for sportmonks retry. Uses `sportmonks.py`. |

### `tests/mappers`

| File | Responsibility and connections |
|---|---|
| `__init__.py` | Initialises the tests.mappers Python package. |
| `test_fixture_mapper.py` | Regression coverage for fixture mapper. Uses `fixture_mapper.py`. |

### `tests/services`

| File | Responsibility and connections |
|---|---|
| `__init__.py` | Initialises the tests.services Python package. |
| `test_gameweek_status.py` | Regression coverage for gameweek status. Uses `gameweek_status_service.py`. |
| `test_season_selection.py` | Regression coverage for season selection. Uses `fixture_service.py`. |

### `tests/utils`

| File | Responsibility and connections |
|---|---|
| `__init__.py` | Initialises the tests.utils Python package. |
| `test_alerts.py` | Regression coverage for alerts. |
