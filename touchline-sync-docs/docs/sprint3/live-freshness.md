# Sprint 3: keep active matches fresh

Prepared 23 Nov 2026 for Leruo. Builds on [Sprint 2](../sprint2/postgres-handoff.md): one transaction per fixture, `COALESCE` for omitted fields, UTC timestamps.

> **Still blocked by the subscription.** The Sportmonks key reports "Football Free Plan" and gets no PSL data (re-checked today). Everything below is measured on free-plan leagues, on production's stored PSL schedule, and in a replay. The live poller can't show real PSL latency until the paid plan is back.

## 3.1 Polling budget

**Measured inputs**

| Input | Value | Source |
|---|---|---|
| Plan allowance | **Free plan: 180 calls per entity per hour.** The paid figure is not yet known | `rate_limit` block on every response (remaining went 180 → 179 after one call). Livescores and fixtures both count against the `Fixture` entity. Each run now records it in `sync.provider_quota` |
| Peak simultaneous PSL fixtures | **8** (final rounds, 15 and 22 May 2027, all at 13:00 UTC); 7 on 4 Nov 2026 | production `football.fixtures` |
| PSL fixtures in a 16-day window | at most **25** | production |
| Pages per response | `/fixtures/multi`: one unpaginated call for up to **50** ids. `/fixtures/between`: honours `per_page=50`, and **`filters=fixtureLeagues:806` works server-side** | probed today on free-plan leagues |
| `between` range limit | 100 days (a full season is rejected) | probed |
| Provider timestamps | Fixtures and events have **none**. `periods[].started/ended` (Unix time) give kickoff, half-time, second-half and full-time times | probed |

**Cadence (set together with the freshness target)**

| Phase | When | Cadence | Freshness target |
|---|---|---|---|
| Pre-match | kickoff −90 min → kickoff | 5 min | lineups committed within 6 min of publication |
| In play, breaks, interruptions | states 2, 3, 4, 6, 9, 11, 16, 18, 19, 21, 22, 23, 25, 26, or past kickoff and not yet live | **every minute** | **score, phase and events committed within 90 s of the provider (p95)** |
| Recently finished | FT/AET/pens/abandoned, up to kickoff +6 h (covers the 4 h settle window) | 15 min | corrections within 16 min |
| Overdue | still "not started" 3–24 h after kickoff | 15 min | n/a |
| Schedule | hourly PSL window (yesterday → +14 days), daily full PSL season | 1 h / 1 day | kickoff changes within 1 h |

One minute is the floor: Netlify's smallest schedule is one minute. Every due fixture shares a single `/fixtures/multi` call, so a tick costs one call whether 1 or 8 matches are on.

**Worst hour on a matchday** (`hourly_call_budget(8, 25)`):

| Item | Calls/h |
|---|---|
| Live tick, 60 × 1 multi call | 60 |
| Schedule window, 1 page | 1 |
| Recovery after a gap | 1 |
| **Nominal** | **62** |
| **With every call retried once** | **124** |

That's 69% of the free plan's 180. The paid allowance is reported as `rate_limit.remaining` after the first paid call, so once the key is fixed `sync.provider_quota` holds the real figure. Before this sprint the hourly job paged every subscribed league in a 16-day window, and the daily job scanned the entire `/fixtures` corpus. Both are now limited to the PSL: one page per hour, and about 4 chunks of 1–2 pages per day.

**Quota reserve.** The hourly and daily jobs skip when fewer than 120 calls remain in the hour, which is two hours of live ticks. The live tick stops only at zero.

## 3.2 Matchday schedule (implemented)

- **Live tick:** new job `live_sync`, run every minute by Netlify `sync-live` (and by APScheduler in the Python worker).
  - The database is the schedule. Each tick loads PSL fixtures from 48 h ago to 90 min ahead, plus any fixture in a live state.
  - It decides what's due using `match_clock.py` / `matchClock.ts`. Both planners are checked against one table of 21 shared cases.
  - It fetches every due fixture in one call and writes each fixture in its own transaction.
- **Phases are stored as they happen:** first half, half-time, second half, extra time, and interruptions (suspended, delayed, interrupted, awaiting updates). State ids come from `/v3/football/states`, fetched today.
- **Reconciliation:** a finished fixture is re-read every 15 min until 6 h after kickoff.
- **Broad jobs:** they keep schedule maintenance and historical catch-up only, now filtered to the PSL.

**Reversals.** The provider may remove an event, for example a goal disallowed by VAR, or a goal deleted and reissued under a new id. The stored row then gets **`withdrawn_at`** instead of being deleted, so the worker needs no DELETE grant. That settles Sprint 1 decision 2 without the grant, and the 4b replay case now passes.

The rule: only a snapshot that contains events can withdraw any. An empty or missing `events` list is still treated as an omission.

## 3.3 Quota and restarts

- **Rate-limit responses:**
  - A 429 is waited out only if the wait is 15 s or less.
  - A longer wait stops the run (`QuotaExhaustedError`) and records `remaining = 0` until the reset. Ticks then skip without making calls until the quota window rolls over, instead of collecting 429s.
  - Only requests that Sportmonks answered count as calls.
- **No competing jobs:** each job type takes the advisory lock from Sprint 2, so the Python worker and Netlify can't run the same job at once. The live tick and the hourly window job can overlap, and that's safe:
  - Writes are idempotent.
  - The revision is a hash of committed content, so a second path writing the same facts creates **no** new revision (tested).
- **Recovery after a gap:** if the last successful tick was 3 or more minutes ago, the next tick re-reads every tracked fixture from authoritative snapshots:
  - any fixture still stored as live
  - any fixture that kicked off in the last 48 h without settling
  - fixtures in the lineup window

  This doesn't use `/livescores/latest`, which only covers 10 seconds. Tested: a 3 h 20 min outage mid-match recovered to the final 2–1 with **one call**.

## 3.4 Measuring the pipeline

V040 adds these, readable by `app_api_user`:

| Object | What it holds |
|---|---|
| `sync.fixture_revisions` | One row each time a fixture's committed content changes. Columns: `revision`, `state_id`, score, active event count, `provider_at` (latest period timestamp), `observed_at` (response received), `committed_at` (last statement before COMMIT), `sync_job_id` |
| `sync.fixture_freshness` | One row per fixture: current `revision`, `last_observed_at`, `last_changed_at` |
| `sync.fixture_revision_latency` | `observe_to_commit` for every revision, and `provider_to_commit` for revisions where the phase changed. Events carry no provider time, so for goals only observation-to-commit is available |
| `sync.live_fixture_health` | For each fixture that should be live: **`stalled`** (not observed for 3 min: a poller problem), **`healthy_quiet`** (observed, nothing changed for 15 min: a scoreless spell), or `healthy` |
| `sync.sync_jobs.api_calls`, `quota_remaining`, `sync.provider_quota` | Calls per run and the allowance left |

Both writers compute revisions with the same database function, `sync.record_fixture_observation()`, so the revision numbers don't depend on which writer ran.

## Demo and exit test

`tests/integration/test_live_freshness.py::test_demo_live_match_with_five_minute_outage` runs Pirates v Chiefs (19781326) through the production tick code on real Postgres, ticking at :37 past each minute from 12:00 to 16:00:

- It uses the real ids and lineups from the Sprint 1 pack.
- The publish times, periods and a 62' goal later disallowed by VAR are **synthetic**, because we never recorded this match live.
- **Ingestion is stopped from 14:13:30 to 14:18:30, across half-time.**

| Change (provider) | Published | Committed (tick) | Latency |
|---|---|---|---|
| Lineups | 12:50:00 | 12:50:37 | 37 s |
| Kickoff | 13:30:00 | 13:30:37 | 37 s |
| Goal 24' Mbuthuma | 13:54:00 | 13:54:37 | 37 s |
| Goal 30' Mmodi | 14:00:30 | 14:00:37 | 7 s |
| **Half-time (during the outage)** | 14:17:00 | **14:18:37, recovery tick** | **97 s** |
| Second half + sub 46' | 14:32:00 | 14:32:37 | 37 s |
| Goal 55' Moremi | 14:41:00 | 14:41:37 | 37 s |
| Goal 62' (3–1) | 14:49:00 | 14:49:37 | 37 s |
| **VAR reversal (back to 2–1)** | 14:51:00 | 14:51:37 | 37 s |
| Full time | 15:21:00 | 15:21:37 | 37 s |

Database processing per fixture: **at most 32 ms**. Latency is bounded by the one-minute cadence, not by the write.

**Quota:** 127 calls over the four hours, at most **55 in one hour** (14:00–15:00, the 5-minute outage included). That's within the budget's 60.

**Committed revision:** **11** = full time, 2–1.
- Active events: 158473513, 158473722, 158475774, 158476429, and the VAR check 9000000202.
- Withdrawn: 9000000201, the disallowed goal.
- This equals the authoritative snapshot (state, score, events, lineups, stats).
- The resume tick's revision 6 (half-time, 1–1) also equals the snapshot at 14:18:37.

The full revision path is 1, 1, 2 (0–0), 2 (1–0), 2 (1–1), **3**, 22, 22 (2–1), 22 (3–1), **22 (2–1)**, 5. The API role reads the same revision and latency from the views.

Other exit tests in the same file:
- a long outage recovering in one call
- the schedule job and the live tick agreeing on the revision
- quiet ticks costing zero calls
- the quota gate
- `stalled` vs `healthy_quiet`
- withdrawal of a reissued goal
- an empty events list never withdrawing anything

The Netlify code was also run against the same database, with a local stand-in for Sportmonks:
- the live tick, including withdrawal
- the PSL-filtered window job
- quota saved
- `api_calls` recorded per job

## What changes for the API

1. **Read events with `withdrawn_at IS NULL`.** Until V040 a reversed or reissued goal stayed in the table forever. Now it's marked. Unfiltered reads still include it.
2. Freshness during play: alert on `sync.live_fixture_health.poll_health = 'stalled'`. Keep the Sprint 2 check (`max(completed_at)` of `fixture_sync`) for the schedule job.
3. Show "as of" with `sync.fixture_freshness.last_observed_at`, and use `revision` to detect that a fixture changed since you last scored it.

```sql
-- Live health (alert if any row is 'stalled')
SELECT sportmonks_fixture_id, state_id, home_score, away_score, poll_health, observation_age
FROM sync.live_fixture_health;

-- End-to-end freshness, last 24 h: p95 provider -> commit on phase changes
SELECT percentile_cont(0.95) WITHIN GROUP (ORDER BY extract(epoch FROM provider_to_commit)) AS p95_s
FROM sync.fixture_revision_latency
WHERE committed_at > now() - interval '24 hours' AND provider_to_commit IS NOT NULL;

-- Quota: calls this hour per job, and what Sportmonks says is left
SELECT job_type, sum(api_calls) FROM sync.sync_jobs
WHERE started_at > now() - interval '1 hour' GROUP BY job_type;
SELECT remaining, resets_at FROM sync.provider_quota WHERE entity = 'Fixture';
```

## Before production

- **Restore the PSL subscription.** Then confirm the paid allowance from `sync.provider_quota`.
- **Run V040 on staging**, then the two integration suites (`pytest tests/integration`). V040 is additive and doesn't touch the objects V039 reconciled. It was verified on a clean V001→V040 install only.
- **Deploy order:** the API adds the `withdrawn_at IS NULL` filter **before** V040 and the new writer code ship.
- **Netlify time limit:** check `sync-daily` against Netlify's scheduled-function limit. The PSL filter makes it much smaller, but its comment assumes a 15-minute budget.
- **Open decision (Sprint 1 #5):** both writers now have the live job. Only one should be scheduled. Netlify is the one running today.
