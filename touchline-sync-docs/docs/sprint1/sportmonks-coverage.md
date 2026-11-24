# Sprint 1: what Sportmonks supplies

Prepared 23 Nov 2026 for Leruo. Evidence comes from the code at `fbf5d5d` and read-only queries against production, run as `sync_worker_user`. Baseline: 29 worker tests pass at `fbf5d5d`. With the replay pack, the count is 39 passed and 1 expected failure, the known gap below.

> **Urgent: PSL data stopped on 18 Nov.** The daily job still runs; it last wrote competitions, seasons and teams on 23 Nov at 00:01 UTC. Since 18 Nov it only receives free-plan leagues: Danish Superliga, Scottish Premiership, Premier League and Bundesliga. The PSL (806), Nedbank Cup, MTN8, Black Label Cup and CAF Champions League were last returned on 18 Nov at 00:01 UTC. PSL fixtures were last written on 18 Nov at 11:00 UTC. The key in the local `.env` reports `Football Free Plan` and gets "no access" for PSL fixture 19781326. If production uses the same key, the subscription lapsed or was downgraded between 11:00 and 12:00 UTC on 18 Nov. Gameweek 10 starts today, 23 Nov, and no PSL facts will arrive until that's fixed. Check the Sportmonks subscription and the `SPORTMONKS_API_KEY` in Netlify.

## 1.1 Active scheduler

| Job | Production owner | Evidence | Overlap prevention | Last success |
|---|---|---|---|---|
| Hourly fixtures | **Netlify** `sync-fixtures` (`@hourly`) | New rows always appear at minute :00 on match days, e.g. hourly from 13:00 to 21:00 UTC on 31 Oct. No `fixture_sync` rows in `sync.sync_jobs` | **None.** Netlify writes no job row and takes no lock | 18 Nov 11:00 UTC, the last PSL fixture write |
| Daily reference and reconciliation | **Netlify** `sync-daily` (00:00 UTC) | Reference rows written at 00:01–00:02 UTC daily. No `full_sync` rows | None | 23 Nov 00:01 UTC, but with no PSL data since 18 Nov |
| News | **Netlify** `sync-news` (every 3 h) | `sync.news_cache.fetched_at` 23 Nov 09:01 UTC | None | 23 Nov 09:01 UTC |
| Python worker (Render) | **Not running against this database** | The worker marks stale jobs failed whenever it starts, but job 9 (`news`) has been stuck at `running` since 29 Jul. The last `sync_jobs` row is a manual job on 8 Aug | Advisory lock plus a job row, in `4c550f7` | Never ran as a scheduler here |

Implications:

- **One writer, but not the planned one.** Each environment does have a single football writer, but it's Netlify, not the worker in `render.yaml`. Deploying Render now would give two writers. Disable the Netlify football schedules at the same recorded checkpoint.
- **Netlify still uses the old season fallback.** `netlify/lib/db.ts` still has it, so it diverges from the `is_current` contract now in Python.
- **Netlify doesn't fill the gameweek gate.** It never writes `sync.gameweek_sync_status`, which has 0 rows.

## 1.2 PSL coverage sheet

Season 2026/27 (Sportmonks 28123) so far: 240 fixtures (69 finished, 7 postponed, 164 not started), 28 gameweeks. Request includes: `participants;scores;events;statistics;lineups;lineups.player`.

| Data | Provider field | ID domain | Granularity | Arrival (as observed by us) | Missing-data behaviour seen |
|---|---|---|---|---|---|
| Fixtures | `fixtures` | `sportmonks_fixture_id` → `football.fixtures.id` | match | Created at season load (e.g. 28 Jul for GW9) | none |
| Teams | `participants[].meta.location` | `sportmonks_team_id` → `teams.id` | match side | season load | none |
| Players | `lineups.player` | `sportmonks_player_id` (events and lineups join on this, not on `players.id`) | person | first lineup they appear in | 296 of this season's lineup rows have no player id |
| Scores | `scores[description=CURRENT]` | n/a | match, live | each hourly run | none |
| States | `state_id` | 1 not started, 2 1st half, 22 2nd half, 5 FT, 7 AET, 8 FT_PEN, 10 postponed, 12 cancelled, 15 abandoned | match, live | each hourly run | Confirm 7, 8 and 22 against `/core/states` |
| Starting XI | `lineups[type_id=11]` | lineup id plus Sportmonks player id | player | Earliest 0.5 h before kick-off (13:00 for 19781326's 13:30 kick-off), median 0.5 h after | Always 22 per finished fixture |
| Bench | `lineups[type_id=12]` | same | player | with the XI | 16–36 per fixture |
| Substitutions | `events[type_id=18]`. **`player_id` = player on, `related_player_id` = player off** | Sportmonks player | player, minute | the hourly run after it happens | All seasons: 44 of 2,466 have no on-player; 46 on-players aren't in that fixture's lineup |
| Goals | `events[type_id=14/15/16]`. `related_player_id` = assist | Sportmonks player | player, minute | the hourly run after it happens | **8 goals this season have no scorer id**; across all seasons, 4 scorers aren't in the lineup |
| Cards and other events | 19 yellow, 20 red, 21 second yellow, 17 missed penalty, 10 VAR | Sportmonks player | player, minute | live | All seasons: 75 of 1,136 yellow cards have no player |
| Team stats | `statistics[]` (EAV: `type_id`, `data.value`) | `sportmonks_stat_id`, **team only** | team per match | Median 0.5 h after kick-off, then updated live | Rows for zero values are often omitted (red cards in 40 of 291 fixtures) |
| Player stats | `lineups.details` (not requested) | n/a | n/a | n/a | **Unsupported in our ingestion**; provider support for the PSL on our plan is **unknown** |
| Availability (injuries, suspensions) | `sidelined` (not requested) | n/a | n/a | n/a | **Unsupported**; manual `app.player_availability_overrides` (V038) is the only source |
| Images | `players.image_path` | n/a | n/a | n/a | All 2,100 point to `cdn.sportmonks.com` (hotlinked, not stored) |

The arrival figures measure our hourly job, not the provider. P90 first-seen latency is 47 h, mostly because of job gaps. Provider latency needs the live probe below.

**Retention and use: open, needs the account holder.** I couldn't find permitted retention of payloads, image hotlinking, or storing replay samples in the repo. The replay pack therefore holds payloads **rebuilt from our stored rows** (real IDs, field shapes the mappers read), not provider responses. Raw probe output stays out of git.

**Schema drift found:** production `football.player_match_stats` has an integer `player_id` column, all NULL in 20,906 rows, that no migration creates. It was a manual change. Add it to the staging diff in the sync-worker plan.

## 1.3 Scoring inputs

| Input | Status | Basis |
|---|---|---|
| Appearance, starter vs bench | **Available** | `player_lineups.type_id` 11/12 |
| Minutes played | **Derivable** | XI plus sub minutes; extra time and injury time are approximate |
| Goals, assists | **Available, with gaps** | Events 14/15 and their `related_player_id`; 8 unattributed goals this season |
| Own goals | Available | event 16 |
| Cards | Available | events 19/20/21 |
| Clean sheet, goals conceded | Derivable | Score plus minutes on pitch, per team |
| **Saves** | **Unsupported per player** | Stat 57 is stored per team only, e.g. Pirates 1, Chiefs 1 in fixture 19781326 |
| **Defensive contributions** (tackles 78, interceptions 100, clearances, blocks) | **Unsupported per player** | Team-level only (Chiefs 20 tackles, Pirates 13); clearances and blocks aren't stored at all |
| Penalty saved or missed | Partial | Event 17 has no attributable keeper |

Each scoring input falls into one of four categories:

- **Confirmed zero:** the fixture is finished and settled, and the stat is a type the source supplies but is absent. For team stats, this assumes absence means zero; confirm that rule.
- **Delayed:** the fixture is finished but inside the 4 h settle window, so `gameweek_sync_status` stays pending.
- **Unknown:** provider support hasn't been verified, e.g. player-level `lineups.details` for the PSL.
- **Unsupported:** not ingested and not planned (saves and defensive contributions per player, availability).

**Before marking saves or defensive contributions available,** fetch one finished PSL fixture with `include=lineups.details` using the **production** key once PSL access is restored. If player-level type 57/78/100 rows come back, it's a mapper and schema change. If not, it's a product decision (drop or rework those scoring rules).

## 1.4 Reference gameweek: GW9, Orlando Pirates 2–1 Kaizer Chiefs

Sportmonks fixture **19781326**, our fixture id **1393**, season 28123 (id 15), teams Pirates 9960 (id 50) and Chiefs 2541 (id 76). Kick-off 31 Oct 13:30 UTC.

| Step | Observed (UTC) | Source | Payload change | Expected DB state |
|---|---|---|---|---|
| 0 | 31 Oct 13:00 | reconstructed | Lineups, state 1 | 6 replay lineup rows (Lungu on the bench), 0 events, no score |
| 1 | 14:00:41 | reconstructed | Goal 158473513, 24', Mbuthuma 37639894, assist Mokobodi 37723175 | 1 goal row, scorer joins to a starter lineup row, 1–0 |
| 2 | 15:00:43 | reconstructed | Mmodi 22860191 30'; sub 158475774 46', Lungu 21395025 on for Ndaba 37708606; Moremi 37551389 55' | 3 goals, 1 sub; the 24' goal keeps its 14:00:41 first-seen time |
| 3 | 16:00 | reconstructed (stat ids synthetic) | State 5, 2–1, team stats | 2 save rows (one per team, no player); gameweek pending until 4 h settle, complete at 18:00 |
| 4a | 1 Nov 23:00 | synthetic | 24' goal re-attributed to Mokobodi, **same** event id | Still 3 goals; the existing row is updated in place and keeps the same internal id |
| 4b | 1 Nov 23:00 | synthetic | Provider deletes 158473513 and reissues the goal under a **new** id | Expected 3 goals; **actual 4**, because the stale Mbuthuma goal stays |

The pack is in `tests/replay/fixture_19781326/` and `tests/replay/test_reference_replay.py`. Each step also replays twice to prove idempotency.

## Demo trace

- **Goal:** 158473513 → `match_events.sportmonks_player_id` 37639894 → `player_lineups` starter (lineup 14675212789) → `players` "Yanela Mbuthuma", FWD. **Traced to player level.**
- **Substitution:** 158475774 → on 21395025 (bench lineup 14675212804), off 37708606 (starter lineup 14675212782). **Traced to player level.**
- **Proposed player stat, goalkeeper saves (type 57):** stored only as team rows, `team_id` 50 and 76. **Coverage gap; needs a product decision** (see 1.3).

## Decisions for Leruo

1. **Postponed fixtures.** The 7 postponed from 4 Nov currently hold their gameweek pending. Ignore them, or move them to the rescheduled gameweek?
2. **Reissued corrections (4b).** Approve a migration granting `sync_worker_user` DELETE on `football.match_events`. Then reconcile per fetched fixture and flag the gameweek for rescore.
3. **Saves and defensive contributions.** Decide after the `lineups.details` probe.
4. **Unattributed goals, 8 this season.** Score them for no one and hold the gameweek pending, or proceed?
5. **Scheduler.** Retire the Netlify football schedules, or keep Netlify and port job tracking and locking to it?
