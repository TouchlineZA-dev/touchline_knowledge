# Sprint 5: recovery and secure operations

Prepared 23 Nov 2026 for Leruo and the on-call. Builds on Sprints 3–4 (live ticks, revisions, completeness). The operating guide is [docs/runbook.md](../runbook.md). This page records what was built and what the exercises proved.

## 5.1 Operational visibility

**`sync.ops_health`** (V049, first written as V042) has one row per check: status (`ok`/`warn`/`alert`), current value, threshold, **named responder**, and runbook section. `app_api_user` can read it, as can `scripts/ops_status.py` and `GET /api/v1/sync/health` (internal key).

| Check | Alert when | Responder |
|---|---|---|
| `active_fixture_observation_age` | an active fixture hasn't been observed for 3 min | Phathu (sync-oncall) |
| `scheduler_heartbeat` | the lease holder's heartbeat is more than 5 min old | Phathu |
| `last_reconciliation` | `full_sync` older than 26 h (alert); `fixture_sync` older than 150 min (warn) | Phathu |
| `quota_headroom` | 120 or fewer `Fixture` calls left (alert); 300 or fewer (warn) | Phathu (provider-account) |
| `job_overlap` | a type running twice, a stuck job, or two schedulers active in 30 min | Phathu |
| `failed_commits` | 3 or more failed jobs in an hour (1–2 is a warn) | Phathu |
| `partial_responses` | due fixtures missing from provider responses (warn) | Phathu (provider-account) |
| `unresolved_ids`, `incomplete_fixtures` | goals or lineup players without ids; incomplete fixtures (warn) | **Leruo** (api-owner) |
| `unknown_provider_states` | a state id we don't know (warn) | Phathu |

The responder names come from `sync.ops_responders`, which only a database admin can edit. **Confirm the names.** I seeded Phathu for the sync and provider roles and Leruo for scoring.

**Delivery:**
- `ops-check` runs on Netlify every 5 minutes and posts new alerts, 30-minute reminders and recoveries to `ALERT_WEBHOOK_URL`, deduplicated in `sync.ops_alerts`.
- `ops-check` can't report Netlify itself being down. For that, each successful live tick pings **`HEARTBEAT_URL`**, an external dead man's switch such as a Healthchecks-style service, which alerts when the pings stop.
- Set both variables in Netlify production.

**Single scheduler.** `sync.scheduler_lease` lets exactly one runner run jobs.
- The other runner stands by and doesn't create job rows.
- Takeover happens only when the holder's heartbeat is more than 10 minutes old.
- `sync_jobs.runner` records who ran each job.
- The Python worker also has `SCHEDULER_ENABLED`, so it can serve manual triggers without scheduling.

## 5.2 Failure exercises

`tests/integration/test_operations.py` runs these on real Postgres with the real roles:

| Exercise | What it showed |
|---|---|
| **Throttling** (429, reset in 25 min) | **one** call, no retry storm; quota recorded as 0; following ticks skip without calls; `quota_headroom` alerts; stored score and state unchanged |
| **Bounded retries** (503 every time) | exactly **5** attempts, then the run fails |
| **Database interruption** (the worker's connections killed between two fixtures) | the first fixture stayed committed; the second **kept its last confirmed state** (state 22, not a partial write); the next run reconciled both |
| **Unknown provider state** (99) | still polled every minute; completeness reads "not finished (state 99)"; `unknown_provider_states` warns; clears when the provider sends a known state |
| **Stopped worker and single-scheduler restart** | a standby with a fresh holder is refused and writes no job row; takeover only after 10 min of silence; the old holder then stands by |

## 5.3 Access and secrets

| Control | Evidence |
|---|---|
| Staging and production credentials are separate | `EXPECTED_DB_PROJECT_REF` is required in staging and production. The worker refuses to start if `DATABASE_URL` points at another project. A production URL copied into staging is rejected (test). Netlify functions have the same guard (smoke-tested) |
| Restricted job controls | `/api/v1/sync/*`, now including `/sync/health`: no key gives 422, a wrong key 401 (constant-time comparison). `/health` reveals no configuration |
| Database permissions | The worker can't rewrite event history or late corrections, edit responders, update gameweeks, or read users. The API role can read every ops view but can't write corrections, take the lease, record observations or change alert state. Plus all Sprint 2 boundaries |
| Log redaction | One filter on every log handler, plus alert payloads and stored `error_message`. It masks configured secrets, URL passwords, `api_token=`, `password=` and `Authorization` values, including tracebacks. Tested in dev and JSON formats; Netlify uses the same rules |
| Provider key stays in the worker | In this repo, only worker code reads `SPORTMONKS_API_KEY` (test). No literal token in tracked files. **The live key and the production worker password appear nowhere in git history** (checked without printing them). Run the same grep in touchline-api and the frontend (runbook §11) |
| Key rotation rehearsal | `scripts/check_provider_key.py` prints a fingerprint, plans, quota and PSL access, never the key. Today it reports fingerprint `bd1295f0`, "Football Free Plan", **PSL access: NO**: the outage is still a subscription problem. Rotation steps are in runbook §11 |

## 5.4 Restore and replay

`scripts/restore_rehearsal.sh`:
1. builds an isolated Postgres 17 (Supabase runs 17.6) from the migrations
2. copies the source's football and sync data plus `app.gameweeks`, only reading the source (no user data)
3. checks every table by row count and content checksum
4. replays Pirates v Chiefs (19781326) and writes a report

| Source | Tables matched | Rows | Replay | Time to recovery |
|---|---|---|---|---|
| Local drill database (full Sprint 3–5 history) | 23/23 | 111 | same internal ids; event history intact; stored hash recomputes; revision 7 → 8 (full time, 2–1); second replay added nothing | **8.0 s** |
| **Production** (read-only dump, 5 MB) | 13/13 | 40,756 | fixture 1393 and all 17 event ids unchanged; full time, 2–1; revision 1 (production has no V040 history yet); second replay added nothing | **54.8 s** (38 s of it the dump over the network) |

Two problems the rehearsal surfaced, both fixed:
- **Checksums depend on server collation.** Postgres 15 on glibc and 17 on musl sort text differently, so the verifier now uses `COLLATE "C"`.
- **The worker role can't read `app.gameweeks_id_seq`.** That's correct after V039, so gameweeks are copied as rows and the sequence is reset afterwards.

## Demo and exit test

`test_runbook_drill_stale_fixture_restart_reconcile_prove` runs every step of the runbook:
- **Staging:** `scripts/stage_incident.py` sets up a second half in which scheduler `netlify` stopped 22 minutes earlier.
- **Operator:** the operator follows the runbook with read-only credentials.

**§1 Diagnose** (`ops_status.py`, as `app_api_user`):

```
ALERT  active_fixture_observation_age   1 of 1 active fixture(s) stalled; oldest observation 00h22m ago
ALERT  scheduler_heartbeat              netlify last heartbeat 00h22m01s ago
  19781326: state 3, 1-1, revision 6, last observed 22m01s ago -> stalled
* No scheduler heartbeat for 22m01s. The active scheduler is 'netlify'. Restart THAT one (runbook §3)
```

**§3 Restart netlify:** its first tick is a recovery tick. **§4 Reconcile:** `reconcile.py` runs the window job and a recovery tick as `operator`.

**§4 Prove it** (`ops_status.py --since <stop time>`):

```
live ticks stopped 12:29:26 -> resumed 12:51:27 UTC (gap 22m01s); first tick after the gap: job #72 by netlify
  19781326 rev 7: state 22, 2-1, 5 events; job #72 (netlify), phase 3 -> 22 published 12:31:26, observed 12:51:27 (delayed 20m01s)
      observed: substitution 46'
      observed: goal 55'
      observed: VAR 63'
```

What it proves:
- **What was delayed:** the second-half start by 20 minutes, the 46' substitution, the 55' goal and the VAR check.
- **The disallowed 62' goal never surfaced:** it was scored and overturned during the gap.
- **Recovery is proven from persisted records only:** job rows, revisions and event history.

The drill also caught a **real bug**: an idle Python tick (nothing due) raised `KeyError: 'revisions'`, so the standby worker would have failed every quiet minute. It's fixed, with a regression test.

**Tests:** 143 Python tests pass on V001→V042 (now V049); 90 pass and 53 skip without a database. 23 TypeScript tests pass and `tsc` is clean. The Netlify lease, runner, `ops-check` and project-ref guard were smoke-tested against the database.

**Still to do with a person:** a second operator, who hasn't seen this work, runs the drill on staging:
1. `stage_incident.py --staging-ref …`
2. `DRILL_OFFSET_SECONDS=… python -m tests.tools.fake_sportmonks`
3. Point a worker at the fake provider and follow `docs/runbook.md` only.

## Before production

1. **Database:** V040–V049 on staging, then `pytest tests/integration`, then production. Before V040, the API must filter `withdrawn_at IS NULL`.
2. **Netlify production context:** set `EXPECTED_DB_PROJECT_REF`, `ALERT_WEBHOOK_URL` and `HEARTBEAT_URL`. Make sure deploy previews and branch deploys have **no** production values.
3. **Responders:** confirm the names in `sync.ops_responders`.
4. **Subscription:** restore the Sportmonks PSL subscription, then do the key-rotation steps once for real.
5. **Volume:** the live tick writes a `sync_jobs` row every minute (about 525k a year). The worker has no DELETE. Agree a retention job run by an admin role.
