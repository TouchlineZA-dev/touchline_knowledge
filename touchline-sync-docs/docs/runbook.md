# Touchline sync runbook

For whoever is on call for the sync worker. Every alert names a section here (`action` in `sync.ops_health`). You need only the read-only steps to diagnose; the write steps say which credentials they use.

**Responders** (`sync.ops_responders`, edited by a database admin):

| Role | Who | Covers |
|---|---|---|
| `sync-oncall` | Phathu | scheduler, jobs, database writes |
| `provider-account` | Phathu | Sportmonks subscription, key, coverage, quota |
| `api-owner` | Leruo | scoring decisions: unresolved ids, incomplete fixtures, rescoring |

**Where things run.** Exactly one scheduler runs the sync jobs at a time. It holds `sync.scheduler_lease`:

| Runner | What it is | How to restart it |
|---|---|---|
| `netlify` (current production) | Scheduled Functions `sync-live` (every minute), `sync-fixtures` (hourly), `sync-daily` (00:00 UTC), `ops-check` (every 5 min) | Netlify → Site → Functions → check the functions are listed and enabled. Then **Deploys → Trigger deploy → Deploy site**. Scheduled functions run only on the published production deploy |
| `python-worker` (standby, Render) | `app.jobs.scheduler`, when `SCHEDULER_ENABLED=true` | Render → service → **Manual Deploy → Restart** |
| `operator` | `scripts/reconcile.py`, run by a person | not a scheduler |

A standby takes over by itself only if the holder's heartbeat is **more than 10 minutes** old. Never run two schedulers on purpose.

---

## §1 Look first (read-only)

```bash
export OPS_DATABASE_URL='postgresql://app_api_user:…@<pooler>:5432/postgres'   # read-only is enough
python scripts/ops_status.py
```

It prints:
- every check, with its responder and section
- the lease holder and its heartbeat age
- the last job of each type and who ran it
- any active fixture that isn't healthy
- a **Diagnosis**

Note the time the trouble started; you need it for §4.

The same checks without the script:

```sql
SELECT * FROM sync.ops_health ORDER BY status = 'ok', check_name;
SELECT * FROM sync.live_fixture_health WHERE poll_health <> 'healthy';
SELECT runner, heartbeat_at, now() - heartbeat_at AS age FROM sync.scheduler_lease;
```

## §2 Stale active fixture

`active_fixture_observation_age` is in alert: a fixture that should be live hasn't been observed for more than 3 minutes. The cause is one of these:

| What §1 shows | Cause | Go to |
|---|---|---|
| `scheduler_heartbeat` alert | the scheduler stopped | §3, then §4 |
| heartbeat ok, `failed_commits` alert | ticks run but fail | §7 |
| heartbeat ok, `quota_headroom` alert | ticks skip for quota | §5 |
| heartbeat ok, `partial_responses` warn | the provider no longer returns the fixture | §8 |

`healthy_quiet` is **not** a problem: the fixture is observed and simply hasn't changed (a scoreless spell).

## §3 Restart the scheduler

1. Find the holder: `ops_status.py` prints "The active scheduler is '<runner>'".
2. Restart **that** runner, using the table above. Don't start the other one: it would stand by anyway until the lease is 10 minutes stale, and two schedulers waste quota.
3. Within 2 minutes, `scheduler_heartbeat` is `ok`, and the last `live_sync` job shows the runner and `success`. The first tick after a gap of 3 minutes or more is a **recovery tick**: it re-reads every active or recently finished fixture.
4. If the holder can't be restarted (the platform is down), start the standby. It takes over once the lease is 10 minutes stale. Record that in the incident notes.

## §4 Reconcile a missed window

After §3, re-read the window the scheduler missed. Use the worker's credentials from `.env` (`DATABASE_URL`, `SPORTMONKS_API_KEY`):

```bash
python scripts/reconcile.py --from 2026-11-22 --to 2026-11-23
```

It runs a PSL `fixture_sync` for the dates, then a recovery `live_sync`. Both are recorded with `runner = operator`. If the scheduler is already running the same job, that step is skipped.

**Prove recovery from persisted records:**

```bash
python scripts/ops_status.py --since <time the trouble started>
```

You should see:
1. `live ticks stopped HH:MM -> resumed HH:MM (gap …)` and the first job after the gap
2. each revision that had been waiting, the events it brought in (`observed: goal 55'` …), and, for phase changes, `published HH:MM, observed HH:MM (delayed …)`
3. any `late correction` the API must rescore
4. no alert left except ones explained elsewhere

Paste that output into the incident notes. It is the record of what was delayed and that it recovered.

If `last_reconciliation` is the alert (the daily job hasn't succeeded in 26 hours), find out why `sync-daily` / `daily_full_sync` failed (§7). Then trigger it manually: Netlify → Functions → `sync-daily` → **Run now**, or `POST /api/v1/sync/full` with `X-Internal-Api-Key` on the Python worker.

## §5 Quota exhausted

`quota_headroom` is in alert: 120 or fewer `Fixture` calls are left this hour.
- The hourly and daily jobs already stand down. The live tick continues until 0, then skips until the reset time shown.
- **Do not** run reconcile or trigger jobs: each call is a 429.
- Wait for the reset, then do §4 if a match was live.
- If it happens on an ordinary matchday, the budget is wrong: check `SELECT job_type, runner, sum(api_calls) FROM sync.sync_jobs WHERE started_at > now() - interval '1 hour' GROUP BY 1, 2` for a job using more than its share, and check that only one runner appears (§6).

## §6 Overlapping or stuck jobs

`job_overlap` covers three cases:
- **Two runners active in 30 minutes:** someone started a second scheduler. Stop the one that isn't the lease holder.
- **A job running too long:** `live_sync` more than 5 min, `fixture_sync` more than 30 min, `full_sync` more than 2 h. The process probably died. The next run of that type marks it failed after 3 h. To unblock sooner, ask a database admin to run `UPDATE sync.sync_jobs SET status='failed', completed_at=now(), error_message='stuck: cleared by <you>' WHERE id=<id>`.
- **The same type running twice:** that shouldn't be possible (advisory lock). Report it to `sync-oncall` with the job ids.

## §7 Failing jobs

`failed_commits` shows the latest error. Error text is redacted: secrets show as `***`.

| Error | Meaning | Do |
|---|---|---|
| `provider returned 0 fixtures … subscription or league coverage lost?` | Sportmonks no longer serves the PSL | `provider-account`: §11 key check, then check the subscription |
| `Sportmonks 401` | key rejected | §11 |
| `Sportmonks quota exhausted` | throttled | §5 |
| connection or `OperationalError` | database unreachable or interrupted | Supabase status page; committed fixtures are safe (one transaction per fixture). When it's back, §4 |
| `DATABASE_URL does not point at project …` | a variable from the wrong environment | fix the deploy-context variables (§11), never the check |

## §8 Partial responses

Due fixtures are missing from provider responses. Check the fixture with `scripts/check_provider_key.py` and in the Sportmonks UI. If the fixture was deleted or moved out of the PSL, nothing is wrong on our side; tell `api-owner`. Stale snapshots refused (`stale_snapshots`) are expected now and then: they're older snapshots arriving late, kept out by design.

## §9 Unresolved ids and incomplete fixtures

These go to `api-owner`. They are scoring decisions, not outages. Details per fixture:

```sql
SELECT sportmonks_fixture_id, status, reason, unresolved
FROM sync.fixture_completeness WHERE status IN ('incomplete', 'unavailable') ORDER BY assessed_at DESC;
```

## §10 Unknown provider state

Sportmonks sent a state id we don't know. The fixture keeps being polled every minute (never dropped), and completeness says `not finished (state N)`. Look the id up with `GET https://api.sportmonks.com/v3/football/states` (25 states as of 24 Nov 2026; `/v3/core/states` returns nothing). Then add it in `app/services/match_clock.py`, `netlify/lib/matchClock.ts`, `tests/fixtures/live_plan_cases.json`, and the state lists in the latest migration's views.

## §11 Keys and credentials

**Separation.** Staging and production use different Supabase projects and different Sportmonks keys.
- Every deploy sets `EXPECTED_DB_PROJECT_REF`. The worker and the Netlify functions refuse to start if `DATABASE_URL` points at another project.
- Netlify: set production values in the **Production** deploy context only. Deploy previews and branch deploys must get the staging values.

**Who holds the provider key.** Only the sync worker: Netlify production context, Render worker, and the on-call's local `.env`. It must never be in touchline-api or the frontend. Check those repos with `git grep -n -i sportmonks_api_key`; it should find nothing.

**Rotate the Sportmonks key** (rehearsed with `scripts/check_provider_key.py`, which prints a fingerprint, never the key):
1. MySportmonks → API tokens → create a new token.
2. `SPORTMONKS_API_KEY=<new> python scripts/check_provider_key.py` should show `PSL access: yes`. Note the fingerprint.
3. Netlify → Site configuration → Environment variables → `SPORTMONKS_API_KEY` → set it for the **Production** context. Render: set it on the worker service if it's used.
4. Trigger a deploy. Within 2 minutes, a `live_sync` or `fixture_sync` job is `success` and `sync.provider_quota.observed_at` is fresh.
5. Delete the old token in MySportmonks. Confirm the next jobs still succeed.
6. Update your local `.env`. Run `python scripts/check_provider_key.py` to see the same fingerprint as step 2.

**Rotate the database role passwords:** `scripts/rotate_role_passwords.py` (Sprint 2). Then update `DATABASE_URL` in every place that uses the role, and the API's connection string for `app_api_user`.

**Job controls.** `/api/v1/sync/*` (trigger full or fixture sync, job status, ops health) needs `X-Internal-Api-Key`. `/health` and `/ready` are open and reveal no configuration.

## §12 Restore and replay

For a real disaster, restore Supabase's backup (point-in-time recovery) **into a new project**, never over the live one. Then verify it the same way as the rehearsal:

```bash
SOURCE_URL='postgresql://sync_worker_user:…@<pooler>:5432/postgres' ./scripts/restore_rehearsal.sh
```

The rehearsal:
1. builds an isolated Postgres 17 from the migrations
2. copies football and sync data (and `app.gameweeks`, no user data) from `SOURCE_URL`, which it only reads
3. compares every table
4. replays fixture 19781326 and checks that ids, history, score and revision behave
5. writes `restore-report.json` with the time to recovery

Exit 0 means it all matched.
