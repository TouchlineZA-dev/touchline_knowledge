# Touchline Sync • Sprints 3–5: live freshness, corrections and operations

[Team home](MASTER.md) · [Football and news sync](touchline_sync.md) · [Database and migrations](DB.md) · [API scoring consumer](touchline_api.md)

**Source • checkout named `touchline`, branch `sprint/3-4-live-freshness-corrections`, commits `7f07050` (Sprints 3–4) and `39c3c57` (Sprint 5), 23 November 2026.** Against `main@07718e1` the branch changes 61 files and adds Flyway migrations **V040–V042** (V042 later renumbered to V049). **Status: built and tested locally. Not merged, not on staging, not in production.** The detailed write-ups in the repository are `docs/sprint3/live-freshness.md`, `docs/sprint4/late-changes.md`, `docs/sprint5/operations.md` and the on-call guide `docs/runbook.md`.

> **Still blocked by the subscription.** The Sportmonks key reports "Football Free Plan" and has no PSL access (re-checked on 23 Nov 2026 with `scripts/check_provider_key.py`). PSL data has not arrived since 18 Nov 2026. All latency figures below come from free-plan leagues, production's stored PSL schedule and a replay of Pirates v Chiefs (fixture 19781326). None of them is real live PSL traffic.

## What each sprint delivered

| Sprint | Question it answers | Migration | Main pieces |
|---|---|---|---|
| 3 • Keep active matches fresh | How quickly does a goal reach the database, and what happens after an outage? | V040 | `live_sync` job every minute, quota tracking, per-fixture revisions, soft withdrawal of reversed events |
| 4 • Explain late changes and incomplete matches | Did a fixture I already scored change? Why can't this fixture be scored? | V041 | Event history, reschedule history, out-of-order guard, per-fixture completeness, late-corrections feed |
| 5 • Rehearse recovery and secure operations | How does the on-call notice a problem, fix it and prove the fix? | V042 | Single-scheduler lease, `sync.ops_health` checks with named responders, alerts, redaction, environment guard, restore rehearsal, runbook |

## Sprint 3 • Live freshness (V040)

**Cadence.** A new `live_sync` job runs every minute: Netlify `sync-live`, plus APScheduler in the Python worker. The database acts as the schedule. Each tick loads PSL fixtures from 48 h ago to 90 min ahead, plus any fixture in a live state. `match_clock.py` and `matchClock.ts` decide which fixtures are due, and both are tested against one shared table of cases (`tests/fixtures/live_plan_cases.json`). Every due fixture goes into **one** `/fixtures/multi` call, so a tick costs one call whether one match is on or eight.

| Phase | Cadence | Freshness target |
|---|---|---|
| Pre-match (kickoff −90 min) | 5 min | lineups within 6 min |
| In play, breaks, interruptions | every minute | score, phase and events committed within **90 s** of the provider (p95) |
| Recently finished (to kickoff +6 h) | 15 min | corrections within 16 min |
| Schedule window / full season | 1 h / 1 day | kickoff changes within 1 h |

**Budget.** The worst matchday hour (8 simultaneous PSL fixtures) costs 62 calls nominally, or 124 if every call is retried once. The free plan allows 180 per entity per hour, and the paid allowance is still unknown. The hourly and daily jobs are now filtered to the PSL server-side (`filters=fixtureLeagues:806` works, which contradicts an older code comment). They stand down when 120 or fewer calls remain. A 429 with a long reset stops the run, and later ticks skip without calling until the reset.

**Recovery.** If the last successful tick was 3 or more minutes ago, the next tick re-reads every tracked fixture from authoritative snapshots. It never relies on `/livescores/latest`, which covers only 10 seconds. In testing, a 3 h 20 min outage recovered with one call.

**Reversals are soft.** An event missing from a *non-empty* snapshot gets `football.match_events.withdrawn_at`. An empty events list never withdraws anything. The worker still has no DELETE grant. **The API must filter `withdrawn_at IS NULL`.**

**New readable objects:** `sync.fixture_revisions`, `sync.fixture_freshness`, `sync.fixture_revision_latency`, `sync.live_fixture_health` (`stalled` vs `healthy_quiet`) and `sync.provider_quota`. Both writers call `sync.record_fixture_observation()`, so revision numbers don't depend on which writer ran.

**Demo.** A replay of Pirates v Chiefs with a 5-minute outage across half-time. Most changes committed 37 s after publication. The half-time change, published during the outage, committed 97 s after publication on the recovery tick. The replay used 127 calls over four hours, at most 55 in one hour. The final revision equals the authoritative snapshot, with the VAR-disallowed goal withdrawn.

## Sprint 4 • Late changes and incomplete matches (V041)

All of this lives in database triggers and functions, so the Python and Netlify writers behave identically.

| Question | Read |
|---|---|
| Can I score this fixture? If not, why not? | `sync.fixture_completeness`: `status`, `reason`, `unresolved`, `inputs`, `assessed_at` |
| Did a fixture I may have scored change? | `sync.late_corrections` (append-only) |
| What happened to this event over time? | `sync.event_changes`: `observed` / `corrected` / `withdrawn` / `reinstated`, with old and new values and `attribution_revision` |
| Was this fixture moved or postponed? | `sync.fixture_schedule_changes`, with old and new kickoff, state and gameweek |

- **Corrections keep the same row.** A changed scorer or assist updates the same event row and increments `attribution_revision`.
- **Out-of-order guard.** `sync.begin_fixture_observation()` runs first in every fixture write. It refuses a snapshot older than the newest one already committed.
- **Lineups.** Missing fields keep their stored values. A row is withdrawn only when the snapshot has a complete XI (11 starters) for that team.
- **Own goals** (type 15; 16 is a penalty, 14 a goal) are recorded on the team that benefits. Corrected 24 Nov 2026: V041 had the ids swapped, fixed in V050 (see below).
- **Reschedules keep the original fixture row.** In testing: scheduled → postponed → rescheduled moved the fixture from GW9 to GW10 under the same fixture id. **The gameweek policy for moved fixtures is the API's decision.**
- **Completeness rule v1.** Statuses are `not_finished`, `settling` (within 4 h of kickoff), `complete`, `incomplete`, and `unavailable` (still missing 72 h after kickoff, or abandoned).

## Sprint 5 • Recovery and secure operations (V042)

**One scheduler at a time.** `sync.scheduler_lease` / `claim_scheduler_lease()`:
- Exactly one runner schedules jobs at a time.
- The standby writes no job rows.
- Takeover happens only after 10 minutes without a heartbeat.
- `sync.sync_jobs.runner` records `netlify`, `python-worker` or `operator`.
- The Python worker also has `SCHEDULER_ENABLED`, so it can serve manual triggers without scheduling.

**Visibility.** `sync.ops_health` has one row per check, each with status, value, threshold, a named responder and a runbook section:

| Check | Alerts when | Responder (seeded, **unconfirmed**) |
|---|---|---|
| `active_fixture_observation_age` | a live fixture hasn't been observed for 3 min | Phathu (sync-oncall) |
| `scheduler_heartbeat` | the lease holder's heartbeat is more than 5 min old | Phathu |
| `last_reconciliation` | `full_sync` older than 26 h | Phathu |
| `quota_headroom` | 120 or fewer calls left | Phathu (provider-account) |
| `job_overlap`, `failed_commits` | jobs overlapping or stuck; 3 or more failures in an hour | Phathu |
| `partial_responses`, `unknown_provider_states` | due fixtures missing from responses; an unknown state id | Phathu |
| `unresolved_ids`, `incomplete_fixtures` | goals or lineup players without ids; incomplete fixtures | Leruo (api-owner) |

Netlify `ops-check` runs every 5 minutes. It posts new alerts, reminders and recoveries to `ALERT_WEBHOOK_URL`, deduplicated in `sync.ops_alerts`. Each successful live tick pings `HEARTBEAT_URL`, an external dead man's switch, which catches Netlify itself being down.

**Security.**
- **Redaction.** One redaction filter covers logs, alerts and stored `error_message` (`app/utils/redact.py`, `netlify/lib/redact.ts`).
- **Environment guard.** `EXPECTED_DB_PROJECT_REF` is required in staging and production, and the worker refuses to start if `DATABASE_URL` points at another project.
- **Job controls.** The `/api/v1/sync/*` routes need `X-Internal-Api-Key`.
- **Database roles.** Permissions were tightened for both the worker role and the API role.
- **Git history.** The live provider key and the production worker password appear nowhere in git history. The same check still needs running in touchline-api and the frontend (runbook §11).

**Failure exercises** (real Postgres, real roles):

| Exercise | Result |
|---|---|
| Throttling | one call, no retry storm |
| Bounded retries | exactly 5 attempts on a persistent 503 |
| Database killed mid-run | committed fixtures stayed; the interrupted one kept its last confirmed state |
| Unknown provider state 99 | still polled; completeness explains it |
| Single-scheduler restart | the standby is refused until the lease is stale |

**Restore rehearsal.** `scripts/restore_rehearsal.sh` builds an isolated Postgres 17 from the migrations. It copies football and sync data plus `app.gameweeks` (no user data, reading the source only), checksums every table and replays fixture 19781326.

| Source | Tables | Rows | Time to recovery |
|---|---|---|---|
| Local drill DB | 23/23 | 111 | 8.0 s |
| Production (read-only dump) | 13/13 | 40,756 | 54.8 s |

**Runbook drill.** `scripts/stage_incident.py` sets up an incident: the scheduler stopped for 22 minutes during the second half. An operator using only `docs/runbook.md` and read-only credentials diagnoses it (§1, `scripts/ops_status.py`), restarts the right scheduler (§3) and reconciles (§4, `scripts/reconcile.py`). They then prove recovery from persisted records with `ops_status.py --since`. The drill also caught a real bug: an idle Python tick raised `KeyError: 'revisions'`. It's fixed, with a regression test.

## Verification on 23 November 2026

- **Python:** 143 tests pass on a clean V001→V042 database; without a database, 90 pass and 53 skip.
- **TypeScript:** 23 tests pass and `tsc` is clean.
- **Netlify smoke test:** the lease, runner, `ops-check` and project-ref guard were smoke-tested against a local database.

Integration tests TRUNCATE tables, so they must never point at production.

## What changes for the API (Leruo)

1. Filter events with `withdrawn_at IS NULL` **before** V040 ships.
2. Score only fixtures whose `sync.fixture_completeness.status = 'complete'`. Watch `sync.late_corrections` for rescoring.
3. Show "as of" from `sync.fixture_freshness.last_observed_at`. Use `revision` to tell whether a fixture changed since it was scored.
4. Decide the gameweek policy for rescheduled fixtures from `sync.fixture_schedule_changes`.

## Before production • open items

| Item | Owner |
|---|---|
| Restore the Sportmonks PSL subscription, then do the key-rotation steps (runbook §11) once for real | provider-account (Phathu) |
| Confirm the responder names in `sync.ops_responders` | team |
| Run V040–V042 on staging, then `pytest tests/integration`, then production | backend |
| API adds the `withdrawn_at IS NULL` filter first | Leruo |
| Set `EXPECTED_DB_PROJECT_REF`, `ALERT_WEBHOOK_URL` and `HEARTBEAT_URL` in the Netlify **Production** context only | backend |
| Agree a retention job for `sync.sync_jobs` (live ticks add about 525k rows a year; the worker has no DELETE) | team / DB admin |
| A second operator runs the runbook drill on staging without help | someone who hasn't seen this work |
| Open a PR from `sprint/3-4-live-freshness-corrections` (all three sprints are on it) | Phathu |
| Decide which single football writer runs per environment. Netlify runs today; the lease prevents two at once but doesn't decide which | team |

## Local checkout note

On Phathu's Mac the `touchline` checkout was inside iCloud Drive. iCloud created duplicate git refs (`refs/remotes/origin/HEAD 2`) and broke syncing. The checkout now lives at `~/Documents/touchline.nosync`, which iCloud skips, with a `~/Documents/touchline` symlink so existing paths still work. Anyone keeping a checkout in an iCloud-synced folder should do the same.

## Sprint 6 follow-up • API handoff fixes (24 Nov 2026)

Branch `sprint/6-api-handoff-fixes` (commit `cf69e84`). The full answers are in `docs/sprint6/api-handoff.md`.

- **V042 is renumbered to V049** so it runs after the API team's V043–V048.
- **V050:**
  - Corrects the goal type ids: 14 is a goal, 15 an own goal, 16 a penalty. Every goal type now needs a scorer, and rescinded goals are ignored (completeness rule v2).
  - A fixture's gameweek comes from `app.fixture_gameweek_assignments` through `sync.fixture_gameweek()`.
  - Adds the `sync.gameweek_completeness` rollup, which replaces `gameweek_sync_status`.
- **V001 no longer contains role passwords.** Run `./migrate.sh repair` once on every existing database before migrating, then rotate both role passwords.
