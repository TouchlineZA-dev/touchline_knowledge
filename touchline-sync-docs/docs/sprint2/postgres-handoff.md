# Sprint 2: Postgres as the handoff

Prepared 23 Nov 2026 for Leruo. Builds on [Sprint 1](../sprint1/sportmonks-coverage.md): reference fixture 19781326, the scheduler inventory, and the PSL outage since 18 Nov.

## 2.1 Database dictionary

### Identity

| Table | Internal key (use in `app.*` FKs) | Provider key (unique) | Joins |
|---|---|---|---|
| `football.fixtures` | `id` | `sportmonks_fixture_id` | `season_id`, `competition_id` and team ids are internal |
| `football.match_events` | `id` | `sportmonks_event_id` | `fixture_id` is internal; players are joined by **`sportmonks_player_id`**, not `players.id` |
| `football.player_lineups` | `id` | `sportmonks_lineup_id` | same as events |
| `football.player_match_stats` | `id` | `sportmonks_stat_id` | team-level; `player_id` is reserved and always NULL today |
| `football.players` | `id` | `sportmonks_player_id` | |
| `sync.sync_jobs` | `id` | n/a | `sync_logs.sync_job_id` |
| `sync.gameweek_sync_status` | `id` | one row per `gameweek_id` | `app.gameweeks.id` |

**Internal IDs are stable.** A provider ID gets its internal `id` the first time we see it, and no upsert ever changes it. All writes go through `INSERT … ON CONFLICT (sportmonks_*_id) DO UPDATE`.

**Correction identity.** A correction that keeps the provider ID updates the existing row in place: same internal `id`, same `created_at`. A correction that deletes the event and reissues it under a new ID currently leaves the old row behind. That's the open decision 2 from Sprint 1, and it needs DELETE on `match_events` for the worker.

### Time

- **All timestamps are `TIMESTAMPTZ` in UTC.** Sportmonks sends `starting_at` as a naive UTC string, and Postgres reads it in the session time zone, which is UTC on Supabase. Don't change the database or role time zone.
- **Gameweek dates are calendar dates in SAST.** `app.gameweeks.start_date` and `end_date` are local dates; a fixture belongs to the gameweek containing its kick-off date in Africa/Johannesburg.
- **`created_at` means first observed by us.** `updated_at` means last written. Every successful run re-saves the fixtures in its window, so `updated_at` changes even when nothing in the row did.

### Meaning of values

| Value | Meaning |
|---|---|
| `fixtures.state_id` | 1 not started, 2/22 in play, 5 FT, 7 AET, 8 FT_PEN, 10 postponed, 12 cancelled, 15 abandoned |
| **Confirmed score** | `home_score`/`away_score` **only when `state_id IN (5,7,8)`**. A non-null score on any other state is provisional; postponed fixture 19781300 stores 0–0 |
| `home_score IS NULL` | no score observed yet |
| A field or collection the provider omits | **The stored value is kept.** Fixture fields are written as `COALESCE(incoming, stored)`, and events, lineups and stats are never deleted. The feed can't reset a value to NULL; a real change such as 1–0 becoming 0–0 after VAR still lands |
| No lineup rows, fixture not started | not published yet; lineups normally arrive around kick-off |
| No lineup rows, fixture finished | **missing**; the provider never supplied them |
| Lineup row with `sportmonks_player_id IS NULL` | the provider listed a player without an ID; this can't be scored |
| No event of a type, fixture finished and settled | none happened (confirmed zero), except for the reissue gap above |
| No stat row of a type | usually zero (the provider omits zero values) but **ambiguous**; don't score from team stats |
| `match_events.rescinded` | `true` disallowed, `false` stands, `NULL` provider didn't say |

### Sync runs, freshness, completeness

| Need | Read |
|---|---|
| Worker alive and getting data | `max(completed_at)` from `sync.sync_jobs` where `job_type='fixture_sync' AND status='success'`. A run that receives none of the fixtures we already hold now **fails**, so this time stops advancing during an outage like 18 Nov |
| Why it stopped | the latest `sync_jobs` row for that type: `status='failed'` plus `error_message`, with detail in `sync_logs` |
| Gameweek ready to score | `sync.gameweek_sync_status.marked_complete`. It's written only after every fixture in the run has committed, and it's never cleared automatically |
| Partially complete | the three component flags: `fixtures_complete`, `events_complete`, `stats_complete` |

## 2.2 Writes are safe to repeat

- **One transaction per fixture** (Python already did this; Netlify did not and now does). The fixture row and all its players, events, stats and lineups commit together, so a reader never sees a new score without its events. A run interrupted mid-fixture publishes nothing for that fixture, and fixtures committed earlier stay valid. Retrying is always safe.
- **The cross-fixture revision is the gameweek gate.** Fixtures commit one by one; `marked_complete` is the signal that the whole set is consistent.
- **Confirmed facts survive partial responses.** These are the `COALESCE` rules for fixtures, with no deletes. This fixed a real bug: a response without `scores` used to erase a confirmed 2–1 to NULL. A test reproduces the old behaviour.
- **One writer at a time, whichever writer runs it.** Python and Netlify now take the same advisory-lock key and write the same `sync_jobs` rows (`fixture_sync`, `full_sync`). Netlify also marks crashed runs stale after 3 h. Netlify now uses the same `is_current` season contract as Python and C#.

## 2.3 Migrations and permissions

**V039** records production's drift and restricts the worker:

- **`player_match_stats.player_id`** and its index are kept as they are, in line with "reuse existing columns where suitable". They're reserved for per-player stats.
- **`football.standings`** is written into the migrations with the definition copied from production.
- **The worker loses all `app` access except `SELECT app.gameweeks`.** Production had granted it SELECT on `app.users`, `fantasy_teams` and `fantasy_squads`, and INSERT/UPDATE on `points_results`, `leaderboards` and `player_prices`. Nothing in this repo uses those, but **confirm nothing in touchline-api or any script connects as `sync_worker_user` for fantasy writes** before V039 reaches production.

Rehearsed locally on Postgres 15 with Flyway (two disposable containers):

| Path | Result |
|---|---|
| Clean install, V001→V039 | 39 migrations applied, all successful |
| Upgrade: V001→V038, then production's drift applied by hand, then V039 | applied; rerunning V039 is a no-op |
| `pg_dump --schema-only` of both, grants and default privileges included | **identical** |

Repeat the same two paths on Supabase staging before production. Local Postgres doesn't have Supabase's own roles and default ACLs.

**Role boundaries,** proved in the integration tests as each real role:

- **`sync_worker_user`** is denied:
  - reading `app.users`, and writing `app.points_results` or `app.gameweeks`
  - DELETE and TRUNCATE on football tables
  - CREATE TABLE
- **`app_api_user`** is denied updating `football.fixtures` and inserting into `sync.sync_jobs`.

**Credentials.** V001's literal passwords are in git history, and the file can't change without breaking checksums. `scripts/rotate_role_passwords.py` sets new passwords from environment variables in one transaction. It sends Postgres only a SCRAM verifier, so DDL logging never sees the plain password. Run it on every new database straight after `./migrate.sh migrate`, and on production if the deployed passwords were ever the V001 ones. Tested locally: the V001 password is rejected afterwards and the new one connects. Operator endpoints (`/api/v1/sync/*`) stay behind `INTERNAL_API_KEY`.

## 2.4 Read examples for the API

Production IDs as of 23 Nov 2026:

| Case | Fixture | Expected result |
|---|---|---|
| Scheduled | `19781298` (id 1379), Durban City v Polokwane City, 24 Nov 17:30 UTC | `state_id=1`, scores NULL, 0 lineup rows (not published), 0 events |
| Confirmed 0–0 | `19781324` (id 1392), Polokwane City v TS Galaxy, 31 Oct | `state_id=5`, 0–0, 40 lineup rows, 11 events (9 subs, 2 yellows), **0 goal events** (confirmed zero) |
| Missing lineup | `19677909` (id 1123), 2025/26, 24 May | `state_id=5`, **0 lineup rows**, so missing; one of 602 earlier-season fixtures like this |
| Lineup with unknown players | `19781321` (id 1390), Kruger United v Richards Bay | 42 lineup rows, some with `sportmonks_player_id IS NULL` |
| Provisional score | `19781300` (id 1380), postponed | `state_id=10` with a stored 0–0, which is **not** confirmed |
| Completed job | none in production yet | The query below returns **NULL** today: no tracked `fixture_sync` has ever succeeded. That correctly reads as "stopped". After the tracking deploys, each run adds a row |

```sql
-- Stopped-worker check (run as app_api_user). Alert if NULL or older than 2 hours.
SELECT max(completed_at) AS last_fixture_refresh
FROM sync.sync_jobs
WHERE job_type = 'fixture_sync' AND status = 'success';

-- Confirmed result for a fixture
SELECT home_score, away_score
FROM football.fixtures
WHERE sportmonks_fixture_id = 19781324 AND state_id IN (5, 7, 8);

-- Scorers, joined to lineups on the provider player id
SELECT e.minute, e.sportmonks_player_id, l.type_id AS lineup_type
FROM football.match_events e
JOIN football.fixtures f ON f.id = e.fixture_id
LEFT JOIN football.player_lineups l
       ON l.fixture_id = e.fixture_id AND l.sportmonks_player_id = e.sportmonks_player_id
WHERE f.sportmonks_fixture_id = 19781326 AND e.type_id IN (14, 15, 16);  -- goal, own goal, penalty
```

## Demo and exit test

`tests/integration/test_postgres_handoff.py` runs against any Flyway-migrated database, using the real worker and API roles. It skips unless all three URLs are set, and it **TRUNCATEs**, so never point it at production.

```bash
export TOUCHLINE_TEST_ADMIN_URL=postgresql://postgres:…@host/db
export TOUCHLINE_TEST_WORKER_URL=postgresql://sync_worker_user:…@host/db
export TOUCHLINE_TEST_API_URL=postgresql://app_api_user:…@host/db
pytest tests/integration -v
```

| Exit criterion | Test |
|---|---|
| Ingest three times; identities and values identical | `test_ingesting_reference_data_three_times_is_stable` |
| One valid fixture/event set from progressive observations | `test_progressive_observations_converge_to_one_valid_set` |
| Interrupt and retry; no partial revision | `test_interrupted_fixture_publishes_nothing_then_retry_completes`, `test_provider_drop_between_fixtures_keeps_committed_ones` |
| Partial responses keep confirmed facts | `test_partial_response_preserves_confirmed_facts`, `test_real_score_change_still_lands` |
| API reads results with its own role | `test_gameweek_status_and_completed_job_visible_to_api` |
| Role boundaries | `test_worker_role_cannot_leave_its_lane`, `test_api_role_cannot_write_football_or_sync` |

All 15 pass locally on a clean V039 install. Still to do on staging: run them there, stop the worker, and have Leruo run the reads above from touchline-api with `app_api_user`.
