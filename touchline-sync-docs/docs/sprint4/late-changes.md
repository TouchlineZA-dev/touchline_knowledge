# Sprint 4: late changes and incomplete matches, explained in Postgres

Prepared 23 Nov 2026 for Leruo. Builds on [Sprint 3](../sprint3/live-freshness.md): authoritative snapshots, one transaction per fixture, a fixture `revision` that changes only when committed content changes.

All of it lives in the database (migration **V041**: triggers and functions), so the Python worker and the Netlify writer behave identically. Both were run against a migrated database.

## What you read

| Question | Table | Key columns |
|---|---|---|
| Can I score this fixture? If not, why not? | `sync.fixture_completeness` (one row per fixture) | `status`, `reason`, `unresolved`, `inputs`, `assessed_at`, `revision`, `completed_revision`, `gameweek_id` |
| Did a fixture I may have scored change? | `sync.late_corrections` (append-only feed) | `fixture_id`, `gameweek_id`, `completed_revision`, `revision`, `changes` |
| What happened to this event over time? | `sync.event_changes` (one row per change) | `change`, `fields`, `old`, `new`, `attribution_revision`, `fixture_revision` |
| Was this fixture moved or postponed? | `sync.fixture_schedule_changes` | `old_/new_starting_at`, `old_/new_state_id`, `old_/new_gameweek_id` |

All four are readable by `app_api_user`.

## 4.1 Event attribution

- **A changed scorer or assist** updates the same event row (same `id`, same `sportmonks_event_id`) and increments **`match_events.attribution_revision`**. The trigger writes a `corrected` row to `sync.event_changes`. That row lists exactly which fields changed, with the old and new values, and the fixture revision the change landed in.
- **Every event's first observation** is also logged (`observed`), so the original attribution can always be traced.
- **A goal removed after review** gets `withdrawn_at` (Sprint 3) and is logged as `withdrawn`. If it comes back, it's logged as `reinstated`.
- **Out-of-order observations:** each fixture write first calls `sync.begin_fixture_observation()`. That locks the fixture and **refuses a snapshot older than the newest one already committed**, for example a slow retry or a second writer racing. Tested: a late-arriving older snapshot could not undo a correction.
- **Partial lineups keep the last confirmed lineup:**
  - Fields missing from a lineup entry, such as a player id or name, keep the stored value.
  - A missing or partial lineup withdraws nothing.
  - A row is withdrawn only when the snapshot has a **complete XI (11 starters) for that team** and the row isn't in it, for example a late team-sheet change.
  - Tested in Python and on the Netlify path: a 6-row snapshot left all 27 confirmed rows in place.

## 4.2 Reschedules keep the fixture

The fixture is keyed on `sportmonks_fixture_id`, so a reschedule updates kickoff and state **on the original row**. A trigger then writes the old and new values, plus the gameweek each one falls in (SAST dates, same rule as the gameweek writer), to `sync.fixture_schedule_changes`.

It records every kickoff change, and every state change into or out of these states:
- postponed, cancelled, TBA, deleted
- delayed
- walkover, abandoned, awarded

Ordinary live progress isn't logged here.

Tested: scheduled → postponed → rescheduled. Fixture id 1 throughout.

| Change | Old → new state | Old → new kickoff | Old → new gameweek |
|---|---|---|---|
| Postponed | 1 → 10 | 31 Oct 13:30, unchanged | GW9 → GW9 |
| Rescheduled | 10 → 1 | 31 Oct 13:30 → 5 Nov 17:30 | GW9 → **GW10** |

The gameweek policy for moved fixtures is yours to apply from these rows.

## 4.3 Completeness, per fixture

`sync.assess_fixture()` runs after every observation (rule version 1) and stores:
- **`inputs`:** what arrived
- **`unresolved`:** what is missing, each with `input`, `issue`, `blocking` and `detail`
- **`assessed_at`:** the assessment time
- **`reason`:** one sentence explaining the status

| Status | Meaning |
|---|---|
| `not_finished` | not started or in play |
| `settling` | finished, within 4 h of kickoff |
| `complete` | every required input is present (below) |
| `incomplete` | something blocking is missing; may still arrive. `reason` says when it becomes `unavailable` |
| `unavailable` | still missing **72 h after kickoff** (treated as permanent), or abandoned |
| `excluded` | postponed, cancelled, TBA or deleted: not played |

Required for `complete`:
1. a final score
2. 11 starters per team
3. at least one event
4. goal events (types 14, 15, 16) per team that **match the score**
5. every goal has a scorer id
6. at least one stats row

Lineup rows without a player id are listed but don't block.

Two rules were checked against production:
- **Own goals** are recorded on the team that benefits. *Corrected 24 Nov 2026:* the original check counted type 16, which is a **penalty**. Sportmonks' types are 14 goal, 15 own goal, 16 penalty (`GET /v3/core/types`). Rechecked on type 15: 16 of 16 production own goals are on the benefiting team, with the scorer in the other team's lineup. V050 also requires a scorer for penalties, which V041 missed.
- **Requiring at least one event** separates a quiet match from a missing feed: a played match always has substitutions or cards. Every one of the 30 finished PSL 0–0s has events.

| Exit case | Result |
|---|---|
| Verified quiet 0–0 | `complete`: "verified quiet 0-0: 22 starters, 2 events, no goal events, 6 stat rows" |
| Incomplete full time, also 0–0 in `football.fixtures` | `incomplete`: "waiting for: lineups missing, events missing, match_stats missing; unavailable from 2026-11-03 13:30 UTC" |
| Missing lineup | `incomplete`, then `unavailable` after 72 h: "still missing 72 h after kickoff, treated as permanently unavailable: lineups missing" |
| Missing stats | `incomplete`, `match_stats missing` |
| Postponed | `excluded`: "postponed: not played; apply the gameweek policy for unplayed fixtures" |
| Goal without scorer | `incomplete`, `events goal_without_scorer`; the unidentified lineup player is listed as non-blocking |

**Real example of `unavailable`:** in production, all 240 finished fixtures of season id 7 have no events, lineups or stats. In the current season all 69 reconcile.

`sync.gameweek_sync_status` is unchanged and still uses the Sprint 1 rules. Use `fixture_completeness` to explain a pending gameweek fixture by fixture.

## 4.4 Late evidence

When a fixture that had been assessed `complete` commits a new revision, the same transaction appends a row to **`sync.late_corrections`**. The row holds:
- `completed_revision`: the revision that was complete, which is what you may have scored
- the new `revision`
- the gameweek
- `changes`: the score or state change, every event change, and every schedule change in that revision

The corrected facts are in the normal tables, and `completed_revision` advances once the corrected fixture reassesses as complete.

**Proposed contract:**
- Store the fixture `revision` you scored with.
- Read `sync.late_corrections WHERE id > <last id processed>`.
- Rescore any fixture whose `revision` is above the one you scored, per the agreed policy.

## Demo

`tests/integration/test_corrections_completeness.py` covers both parts.

**Part 1** (`test_demo_incomplete_full_time_vs_verified_quiet_match`). Two finished 0–0 fixtures have identical rows in `football.fixtures`. `fixture_completeness` separates them, as in the table above.

**Part 2** (`test_demo_assist_reassigned_after_completion`). Pirates 2–1 Chiefs is assessed complete at revision 1. Then the 24' goal's assist moves from Mokobodi (37723175) to Ndaba (37708606), and the score stays 2–1. The same corrected snapshot is then observed again.

- **Visible once:** one event row, the same `id` 1, `attribution_revision` 2, assist Ndaba. Re-observing adds nothing.
- **Original traceable:**

  | Change | Fields | Assist (old → new) | `attribution_revision` | Fixture revision |
  |---|---|---|---|---|
  | observed | — | — → 37723175 | 1 | 1 |
  | corrected | `sportmonks_related_player_id` | 37723175 → 37708606 | 2 | 2 |

- **Fixture to recalculate:** exactly one `late_corrections` row. Fixture 19781326, gameweek 1 (GW9), `completed_revision` 1 → `revision` 2. Its `changes` are `fixture: {}` (scoreline unchanged) plus one event entry with `fields: ["sportmonks_related_player_id"]`.

Also tested: a scorer change keeps the event's identity; a goal ruled out after completion produces a late correction with `score: ["2-1","1-1"]` and the event `withdrawn`; settling, then complete, with no false late correction.

**Results:** 107 Python tests pass on a fresh V001→V041 database, including all Sprint 2 and 3 tests. 23 TypeScript planner tests pass, and `tsc` is clean.

## Before production

1. **Run V041 on staging, then `pytest tests/integration`.** V041 adds triggers to `football.match_events` and `football.fixtures`. They fire on every upsert but write only when something changed.
2. **Backfill.** Existing fixtures get a `fixture_completeness` row the next time they're observed. For the current season that means within a day, through the daily job. Past seasons need a one-off pass if you want them assessed.
3. **Decide on the API side:**
   - accept the completeness rules and the 72 h "permanently unavailable" deadline
   - the rescore policy that `late_corrections` feeds
   - the gameweek policy for postponed or moved fixtures (Sprint 1 decision 1)
