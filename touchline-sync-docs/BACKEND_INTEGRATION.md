# Touchline — Backend Integration Guide

For the developer building the **user-facing backend** (users, fantasy teams,
scoring, leaderboards). This is everything you need to code your part against the
shared database: architecture, connection, every table, the Sportmonks codes,
the join rules, and worked examples.

Last generated from the live DB + Sportmonks reference on 2026-08-08.

---

## 1. The big picture — how the pieces fit

```
 Sportmonks API ─┐
                 ├─► SYNC WORKER (Netlify functions)  ──writes──►  Supabase Postgres
 GNews API ──────┘   (not yours; already running)                 ┌───────────────────────┐
                                                                  │ football  (sync-owned)│  ← you READ
 YOUR BACKEND  ◄──────────────── reads football + sync ──────────►│ sync      (sync-owned)│  ← you READ
 (users, teams,                                                   │ app       (YOURS)     │  ← you READ/WRITE
  scoring, API) ─────────────── reads/writes app ────────────────►└───────────────────────┘
        │
        └──► Frontend
```

**The integration point is the database, not HTTP.** The sync worker keeps
`football` (raw match data) and `sync` (pipeline status) fresh. You read those,
run the fantasy game logic, and write your results into the `app` schema. You and
the sync worker never call each other — you share Postgres.

### Who owns what

| Schema | Owner | You can |
|---|---|---|
| `football` | sync worker | **READ only** — competitions, seasons, teams, players, fixtures, events, stats, lineups, standings |
| `sync` | sync worker | **READ only** — job status + the gameweek-completeness gate |
| `app` | **you** | **READ + WRITE** — users, fantasy teams, squads, gameweeks, points, leaderboards, prices |

This split is enforced by database GRANTs — you connect as a role that physically
cannot write `football`/`sync`, so you can't corrupt match data and the sync
worker can't corrupt your game state.

---

## 2. Connecting to the database

Connect to Supabase Postgres as **`app_api_user`** (get the password from the
repo owner via a password manager — never in code/git):

```
Host=aws-0-eu-west-3.pooler.supabase.com
Port=6543                         # transaction pooler — use this for a web backend
Database=postgres
Username=app_api_user.exyzbdbitrsidompguda
Password=<from password manager>
SSL Mode=Require
```

- **Port 6543** = Supabase transaction pooler (right for many short web requests).
  Port 5432 is the session pooler (fine for long-lived/migration connections).
- Your role: `SELECT` on all of `football` + `sync`; `SELECT/INSERT/UPDATE` on all
  of `app`. No `DELETE` is granted anywhere (by design — soft-delete with flags).

---

## 3. The tables you READ (football schema)

All times are **UTC** (`timestamptz`). Convert to SAST (UTC+2, no daylight saving)
for display: `... AT TIME ZONE 'Africa/Johannesburg'`.

### football.players — the master player list
| Column | Type | Notes |
|---|---|---|
| `id` | int PK | **internal id — this is what app tables reference** |
| `sportmonks_player_id` | bigint | Sportmonks id — what `match_events`/`lineups` reference |
| `firstname`, `lastname`, `display_name`, `common_name` | varchar | names |
| `country_id`, `nationality_id` | int | Sportmonks country ids |
| `position_id`, `detailed_position_id` | smallint | Sportmonks position codes (24=GK,25=DEF,26=MID,27=FWD — see §6) |
| `date_of_birth` | date | |
| `height`, `weight` | smallint | cm / kg (often null) |
| `image_path` | text | photo URL — ~45% are `.../placeholder.png` |
| `gender` | varchar | |

### football.teams
`id` (PK), `sportmonks_team_id`, `name`, `short_code`, `country_id`, `venue_id`,
`founded`, `type`, `image_path` (crest URL), `is_placeholder`, `last_played_at`.

**`short_code` is NULL for 5 of the 16 2026/27 PSL clubs:** Sportmonks doesn't supply one.
Show the proposed code below as a display fallback in the API (`COALESCE(t.short_code, <map>)`).
Don't write them into `football.teams`: the worker overwrites provider fields on every sync.
Codes follow the first three letters, as the provider does for the other 11 (AMA, CHI, GOL, KAI, MSU, GAL, ORL, RIC, SEU, STE, TSG).

| `sportmonks_team_id` | Club | Proposed code |
|---|---|---|
| 274102 | Durban City | `DUR` |
| 274104 | Kruger United | `KRU` |
| 270332 | Milford | `MIL` |
| 12946 | Polokwane City | `POL` |
| 277354 | Siwelele | `SIW` |

*Proposed 24 Nov 2026, not yet agreed. Key on `sportmonks_team_id`: internal ids differ between environments.*

### football.competitions
`id` (PK), `sportmonks_competition_id`, `name`, `short_code`, `country_id`,
`type`, `sub_type`, `active`, `image_path`, `category`.
Five are tracked: Premier League (PSL), CAF Champions League, Nedbank Cup, MTN8
Cup, Black Label Cup.

### football.seasons
`id` (PK), `sportmonks_season_id`, `competition_id` → competitions.id, `name`
(e.g. "2026/2027"), `starting_at`, `ending_at`, `is_current`, `finished`,
`pending`. **Use `is_current = true` to find the live season per competition.**

### football.fixtures — matches
| Column | Type | Notes |
|---|---|---|
| `id` | int PK | internal id — events/stats/lineups reference this |
| `sportmonks_fixture_id` | bigint | |
| `competition_id`, `season_id` | int | → competitions.id / seasons.id |
| `home_team_id`, `away_team_id` | int | → teams.id (**nullable** — TBD/placeholder fixtures) |
| `home_score`, `away_score` | smallint | **null until played**; final score when FT |
| `home_penalty_score`, `away_penalty_score` | smallint | shoot-outs |
| `state_id` | smallint | match status — see §6 (1=Not Started, 5=Full Time, …) |
| `starting_at` | timestamptz | kickoff (UTC); may be provisional pre-season |
| `stage_id`, `venue_id`, `leg`, `length`, `result_info`, `is_placeholder` | | |

### football.match_events — goals, cards, subs (**core scoring data**)
| Column | Type | Notes |
|---|---|---|
| `id` | int PK | |
| `sportmonks_event_id` | bigint | |
| `fixture_id` | int | → fixtures.id (internal) |
| `team_id` | int | → teams.id (internal, nullable) |
| `sportmonks_player_id` | bigint | **Sportmonks id → join to players.sportmonks_player_id** |
| `sportmonks_related_player_id` | bigint | assisting/second player (Sportmonks id) |
| `player_name`, `related_player_name` | varchar | fallback text if the id doesn't resolve |
| `type_id` | smallint | **what happened** — 14=Goal, 19=Yellowcard… (see §6) |
| `minute`, `extra_minute` | smallint | |
| `result` | varchar | e.g. score after a goal |
| `on_bench` | bool | |

### football.player_lineups — who played
| Column | Type | Notes |
|---|---|---|
| `id` | int PK | |
| `sportmonks_lineup_id` | bigint | |
| `fixture_id` | int | → fixtures.id |
| `team_id` | int | → teams.id (nullable) |
| `sportmonks_player_id` | bigint | **→ players.sportmonks_player_id** (~1% null) |
| `jersey_number`, `position_id` | smallint | |
| `type_id` | smallint | lineup type (starter vs bench) |

### football.player_match_stats — ⚠️ TEAM-level stats (misnamed)
Despite "player" in the name, **each row is a team-level statistic for a fixture**,
keyed by `type_id`. There is a `player_id` column but the sync leaves it NULL
(the Starter plan gives no per-player stats).
`fixture_id` → fixtures.id, `team_id` → teams.id, `type_id` (see §6 stat codes),
`value` (the number), `location` ('home'/'away').

### football.standings — league table
`season_id`, `team_id`, `position`, `points`, `played`, `won`, `drawn`, `lost`,
`goals_for`, `goals_against`, `goal_difference`, plus `raw` (jsonb). Handy for
display; not needed for scoring.

---

## 4. The tables you READ (sync schema)

### sync.gameweek_sync_status — **your "is it safe to score?" gate**
One row per gameweek. The sync worker flips these flags as data lands.
`gameweek_id` → app.gameweeks.id (UNIQUE — exactly one row per gameweek),
`fixtures_complete`, `events_complete`, `stats_complete`, `marked_complete`
(all bool). **Only calculate final points for a gameweek once `marked_complete`
(or the flags you care about) is true** — otherwise you're scoring partial data.

*(Note: the sync worker doesn't yet populate this table — it's the agreed
handshake for later. For now, gate on fixture `state_id = 5` (Full Time) per
match instead.)*

### sync.news_cache — SA football news (for a news feed / widget)
The sync worker pulls South African football RSS feeds every 3 hours and stores
the latest ~50 articles here as **one row**: `cache_key = 'sa_football_news'`,
with a JSON array in the `response` (jsonb) column. Read it like:
```sql
SELECT jsonb_array_elements(response) AS article
FROM sync.news_cache WHERE cache_key = 'sa_football_news';
-- each article: { source, title, url, image_url, published_at, summary }
```
Sources: SoccerLaduma, iDiski Times, KickOff, Google News (PSL). `image_url` and
`published_at` can be null. `fetched_at`/`expires_at` tell you freshness.
*(This table is also the generic cache the old GNews path used — filter by
`cache_key` to get the SA-news row.)*

### sync.sync_jobs / sync.sync_logs
Audit trail of sync runs (`job_type`, `status`, timestamps). Read-only; useful
for a health/status page. Not needed for the game.

---

## 5. The tables you OWN (app schema) — read/write

### app.users
`id` PK, `email` (unique), `username` (unique), `password_hash`, `is_active`,
timestamps. **Your auth lives here.**

### app.gameweeks — **you create these**
`id` PK, `name` (e.g. "Gameweek 1"), `start_date`, `end_date`, `is_active`,
`season_id` → football.seasons.id (nullable for now), timestamps.
You define the gameweek calendar (which fixtures belong to which gameweek).

### app.fantasy_teams
`id` PK, `user_id` → users.id, `season_id` → football.seasons.id, `name`,
`budget_remaining` NUMERIC(5,1), default **100.0** (R100m; V035 changed it from V025's 300.0). R100m vs R300m is still awaiting sign-off in the team decisions table, timestamps.
`UNIQUE(user_id, season_id)` — one team per user per season.

### app.fantasy_squads — a team's picks for a gameweek
`id` PK, `fantasy_team_id` → fantasy_teams.id, **`player_id` → football.players.id
(internal id!)**, `gameweek_id` → gameweeks.id, `is_captain`, `is_vice_captain`,
`is_on_bench`, `purchase_price` NUMERIC(5,1), timestamps.
**DB enforces** (V031): a player appears once per team per gameweek; at most one
captain and one vice-captain per team per gameweek.

### app.points_results — **where your scoring engine writes**
`id` PK, `fantasy_team_id`, `player_id` → players.id, `gameweek_id`, `points` int,
`breakdown` jsonb (store the per-event points math here), timestamps.

### app.leaderboards
`id` PK, `gameweek_id`, `user_id`, `fantasy_team_id`, `total_points`, `rank`,
timestamps.

### app.player_prices — current price per player per season
`id` PK, `player_id` → players.id, `season_id`, `price` NUMERIC(5,1) default 50.0,
timestamps. `UNIQUE(player_id, season_id)`.

### app.player_price_changes — append-only price history
`id` PK, `player_id`, `season_id`, `gameweek_id`, `old_price`, `new_price`,
`created_at`. **Never UPDATE/DELETE — insert a new row per change.**

---

## 6. Sportmonks codes (the numbers you'll decode)

These are numeric codes from Sportmonks. Below are the exact ones present in the
PSL/cup data — decode them in your scoring logic.

### Event `type_id` (football.match_events) — for scoring
| id | meaning | fantasy relevance |
|---|---|---|
| **14** | Goal | + (by position) |
| **15** | Own Goal | − |
| **16** | Penalty (scored) | + |
| **17** | Missed Penalty | − |
| **18** | Substitution | appearance / minutes proxy |
| **19** | Yellowcard | − |
| **20** | Redcard | − |
| **21** | Yellow/Red card (2nd yellow) | − |
| **10** | VAR | info only |

> Assists are **not** reliably provided as events on this plan — `related_player`
> is sometimes the assist, sometimes not. Don't build core scoring on assists.

### Fixture `state_id` (football.fixtures) — match status
All 25 states, from `GET https://api.sportmonks.com/v3/football/states` (fetched 24 Nov 2026; there is no 24).

| id | meaning | code | played? |
|---|---|---|---|
| **1** | Not Started | `NS` | not yet |
| **2** | 1st Half | `INPLAY_1ST_HALF` | live |
| **3** | Half Time | `HT` | live |
| **4** | Break | `BREAK` | live |
| **5** | Full Time | `FT` | ✅ final |
| **6** | Extra Time | `INPLAY_ET` | live |
| **7** | After Extra Time | `AET` | ✅ final |
| **8** | After Penalties | `FT_PEN` | ✅ final |
| **9** | Penalties | `INPLAY_PENALTIES` | live |
| **10** | Postponed | `POSTPONED` | no: excluded from completeness |
| **11** | Suspended | `SUSPENDED` | live, interrupted (still polled every minute) |
| **12** | Cancelled | `CANCELLED` | no: excluded |
| **13** | To Be Announced | `TBA` | not yet: excluded until a date is set |
| **14** | Walk Over | `WO` | result awarded, no play |
| **15** | Abandoned | `ABANDONED` | no: competition ruling (unavailable) |
| **16** | Delayed | `DELAYED` | live, interrupted (still polled every minute) |
| **17** | Awarded | `AWARDED` | result awarded, no play |
| **18** | Interrupted | `INTERRUPTED` | live, interrupted (still polled every minute) |
| **19** | Awaiting Updates | `AWAITING_UPDATES` | live, interrupted (still polled every minute) |
| **20** | Deleted | `DELETED` | no: excluded |
| **21** | Extra Time - Break | `EXTRA_TIME_BREAK` | live |
| **22** | 2nd Half | `INPLAY_2ND_HALF` | live |
| **23** | ET - 2nd Half | `INPLAY_ET_2ND_HALF` | live |
| **25** | Penalties - Break | `PEN_BREAK` | live |
| **26** | Pending | `PENDING` | live, interrupted (still polled every minute) |

**Score a match only when `state_id ∈ (5, 7, 8)`**, or better, when `sync.fixture_completeness.status = 'complete'` (V041/V050), which also accepts walkovers and awarded results (14, 17) with no lineups or events. Treat 10/12/13/20 as "not played" and 15 (abandoned) as "needs a ruling".

### Stat `type_id` (football.player_match_stats) — team-level
Common ones: 34 Corners · 42 Shots Total · 86 Shots On Target · 45 Ball
Possession % · 52 Goals · 56 Fouls · 84 Yellowcards · 83 Redcards · 57 Saves ·
79 Assists (team total) · 51 Offsides. (Full list of 44 lives in the sync repo /
Sportmonks `GET /v3/core/types`.)

### Position ids (players / lineups)
`position_id`: **24 = Goalkeeper, 25 = Defender, 26 = Midfielder, 27 = Attacker**.
`detailed_position_id` is finer-grained (e.g. specific roles).

> Full authoritative names for any code: `GET https://api.sportmonks.com/v3/core/types`
> (event + stat types) and `GET /v3/football/states` (match states).

---

## 7. ⚠️ The join rule you MUST get right (mixed ID spaces)

There are **two id spaces**, and mixing them silently returns nothing:

- **`app` tables** (`fantasy_squads`, `points_results`, `player_prices`) reference
  **`football.players.id`** — the *internal* id.
- **`match_events` / `player_lineups`** reference **`players.sportmonks_player_id`**
  — the *Sportmonks* id. (They are NOT foreign keys — events can name players not
  yet in the players table.)

So to link **a fantasy squad pick → their match events**, you bridge through the
`players` table:

```
fantasy_squads.player_id  =  players.id
players.sportmonks_player_id  =  match_events.sportmonks_player_id
```

Never join `fantasy_squads.player_id` directly to
`match_events.sportmonks_player_id` — different id spaces, zero rows.

---

## 8. How to build the fantasy core

The raw material is in `football`; you turn it into points and write `app`.

1. **Gameweeks (you define):** create `app.gameweeks` rows and decide which
   fixtures fall in each (by `starting_at` date range or by round).
2. **Squad selection:** validate against `budget_remaining`, squad size, formation,
   max-per-club — the DB already guarantees no duplicate player and one
   captain/vice per gameweek.
3. **Scoring (per finished match):**
   - Find fixtures in the gameweek with `state_id ∈ (5,7,8)`.
   - For each squad player, pull their `match_events` (via the §7 bridge):
     goals (14) and penalty goals (16) + by position, cards (19/20/21) −, own goals (15) −,
     penalties missed (17) −. An own goal's `team_id` is the team that BENEFITS; the scorer
     (`sportmonks_player_id`) plays for the other side. Skip events with `rescinded = true`
     or `withdrawn_at IS NOT NULL`.
   - **Appearance:** if the player is in `player_lineups` for the fixture →
     appearance points. (No minutes on this plan, so appearance is your proxy.)
   - **Clean sheets (team-level):** the player's team conceded 0 →
     `fixtures.home_score/away_score = 0` for the opponent, awarded to GK/DEF who
     appeared.
   - **Captain:** double the captain's points; if the captain didn't play, promote
     the vice-captain (your rule).
   - Write each result to `app.points_results` (put the math in `breakdown` jsonb).
4. **Leaderboards:** aggregate `points_results` per team per gameweek into
   `app.leaderboards` with a `rank`.
5. **Pricing:** seed `app.player_prices`; log every change to
   `app.player_price_changes` (append-only).

---

## 9. Data caveats (read before you trust a field)

- **Times are UTC.** Convert to SAST for users.
- **No per-player stats** (assists/minutes/ratings) on the Starter plan. Build
  scoring from **events + lineups + team stats**, not from `player_match_stats`
  per player.
- **`player_match_stats` is team-level** despite its name; `player_id` is NULL.
- **Fixtures can have null teams / no score** (postponed, TBD opponents, cancelled).
  Always handle `home_team_id`/`away_team_id`/`home_score` being null.
- **Some event player refs don't resolve** to a `players` row (~a handful per
  season, by design). Fall back to `player_name`.
- **~45% of players have a placeholder image** — supply an avatar fallback.
- **Scores/state are only final at `state_id ∈ (5,7,8)`** — don't score live/NS.

---

## 10. Worked example queries

**Current PSL season fixtures for a gameweek window (SAST):**
```sql
SELECT f.id, (f.starting_at AT TIME ZONE 'Africa/Johannesburg') AS kickoff_sast,
       h.name AS home, a.name AS away, f.home_score, f.away_score, f.state_id
FROM football.fixtures f
JOIN football.seasons s   ON s.id = f.season_id AND s.is_current
JOIN football.competitions c ON c.id = s.competition_id AND c.sportmonks_competition_id = 806
LEFT JOIN football.teams h ON h.id = f.home_team_id
LEFT JOIN football.teams a ON a.id = f.away_team_id
WHERE f.starting_at::date BETWEEN :gw_start AND :gw_end
ORDER BY f.starting_at;
```

**A squad player's goals in a finished fixture (the §7 bridge in action):**
```sql
SELECT p.display_name, count(*) AS goals
FROM app.fantasy_squads fs
JOIN football.players p ON p.id = fs.player_id                    -- app -> players (internal id)
JOIN football.match_events e
     ON e.sportmonks_player_id = p.sportmonks_player_id           -- players -> events (sportmonks id)
JOIN football.fixtures f ON f.id = e.fixture_id AND f.state_id IN (5,7,8)
WHERE fs.gameweek_id = :gw AND e.type_id = 14                     -- 14 = Goal
GROUP BY p.display_name;
```

**Clean-sheet check for a team in a fixture:**
```sql
SELECT CASE WHEN :team_id = f.home_team_id THEN f.away_score = 0
            WHEN :team_id = f.away_team_id THEN f.home_score = 0 END AS clean_sheet
FROM football.fixtures f WHERE f.id = :fixture_id AND f.state_id IN (5,7,8);
```

**Who started/played in a fixture:**
```sql
SELECT p.display_name, l.jersey_number, l.position_id
FROM football.player_lineups l
JOIN football.players p ON p.sportmonks_player_id = l.sportmonks_player_id
WHERE l.fixture_id = :fixture_id;
```

---

## 11. Rules of the road

- **You own `app`; never write `football`/`sync`** (your role can't anyway).
- **Schema changes go through Flyway** (the sync repo's `db/migrations/` +
  `migrate.sh`) — coordinate `app`-schema changes with the repo owner. Don't
  `CREATE TABLE` ad hoc; add a migration.
- **Append-only tables:** `app.player_price_changes`, `sync.sync_logs` — insert
  only, never update/delete.
- **Idempotency:** the sync worker refreshes `football` continuously; treat it as
  live source data that can change (scores update, fixtures move). Recompute
  rather than assuming immutability until a match is final.

Questions on any field or code → ping the repo owner. Full schema truth is
`db/migrations/` in the sync repo.

---

## 12. Images — how photos, crests & logos are served

**Only URLs are stored, never the image files.** Every `image_path` column
(`football.players`, `football.teams`, `football.competitions`) holds a **text
URL** pointing at Sportmonks' CDN. The image bytes never touch this database or
your backend — you just pass the string through and the browser fetches it.

```
Sportmonks CDN ──(sync saves the URL)──► football.*.image_path ──(you return the URL)──► frontend <img src="...">
```

**Real examples from the data:**

| Asset | `image_path` value |
|---|---|
| Player photo | `https://cdn.sportmonks.com/images/soccer/players/5/23223429.png` |
| Team crest | `https://cdn.sportmonks.com/images/soccer/teams/22/28406.png` |
| Competition logo | `https://cdn.sportmonks.com/images/soccer/leagues/6/806.png` |

**Placeholders — you must handle these.** When Sportmonks has no image it returns
a stock URL. Detect the string and substitute your own asset:

| Asset | Placeholder URL contains |
|---|---|
| Player | `.../placeholder.png` |
| Team | `.../team_placeholder.png` |

A large share are placeholders — of ~2,000 players, only ~30% currently have a
real photo (higher for well-known clubs). **Plan a default avatar / generic crest
fallback in the UI**, e.g.:
```js
const isPlaceholder = url == null || url.includes("placeholder");
const src = isPlaceholder ? "/assets/avatar-fallback.png" : url;
```

**Notes:**
- `players.image_path` is currently never null (always at least a placeholder),
  but code defensively — team/competition logos *can* be null.
- **URLs are stable, but content improves over time.** A placeholder today can
  become a real photo later; the sync refreshes `image_path` on every run, so
  just re-read from the DB — no action needed on your side.
- Loading cross-origin via a normal `<img>` tag is fine (no CORS setup needed).
- **Optional — re-hosting:** if you don't want a hard runtime dependency on
  Sportmonks' CDN (or want uniform sizes / your own caching), download each image
  once, store it in **Supabase Storage** or your own CDN, and save *that* URL
  instead. This is an extra background job and is **not required to launch** — the
  Sportmonks CDN URLs work directly.

---

## 13. How the sync worker and your backend run in tandem

Two independent services, one shared database, **zero direct calls between them.**
This is the runtime picture: who acts when, and how a gameweek flows through both.

### The two roles
- **Sync worker** (Netlify Scheduled Functions — already running): the *data
  plane*. On a schedule it pulls Sportmonks and keeps `football` + `sync` current.
  It knows nothing about users, teams, or points. It only writes `football`/`sync`.
- **Your backend**: the *game + application plane*. Serves the API, owns users and
  fantasy state, turns match data into points. Reads `football`/`sync`, writes `app`.

### The golden rule: coordinate through the database, not HTTP
Neither service calls the other — the **schemas are the contract**:
`sync worker → writes football, sync` · `backend → reads football/sync, writes app`.
Keep that boundary and the two deploy, restart, and scale independently.

### A gameweek, end to end

```
BEFORE THE ROUND
  Backend:      create app.gameweeks rows; open squad selection
  Sync worker:  (already) has the season's fixtures in football.fixtures

DURING MATCHES  (every hour, automatically)
  Sync worker:  updates fixtures (scores, state_id); inserts match_events,
                player_lineups, team stats
  Backend:      read live data for display; DO NOT finalise points yet

AFTER A MATCH FINISHES  (state_id in 5/7/8)
  Sync worker:  writes final score + full events/lineups on its next run
  Backend:      detect finished fixtures -> compute points from
                match_events + player_lineups -> write app.points_results
                -> aggregate app.leaderboards

BETWEEN ROUNDS
  Backend:      apply price changes (app.player_price_changes); open next GW
  Sync worker:  keeps syncing; new gameweeks are the backend's concern, not its
```

### The "safe to score" handshake
Because the sync is hourly, at any instant a match may be **half-synced**. Two
ways to know it's safe to finalise points:
1. **Per-fixture (use this now):** only score fixtures with `state_id IN (5,7,8)`.
2. **Per-gameweek (future):** the sync worker will set `sync.gameweek_sync_status`
   flags (`fixtures_complete`, `events_complete`, `marked_complete`) once a whole
   gameweek's data has landed — gate your scoring job on `marked_complete = true`.
   *(Not populated yet — use option 1 until it is.)*

### Responsibility matrix

| Concern | Sync worker | Your backend |
|---|---|---|
| Fixtures, scores, events, lineups, team stats | **writes** | reads |
| Players, teams, competitions, seasons | **writes** | reads |
| Users, auth | — | **writes** |
| Fantasy teams, squads, transfers | — | **writes** |
| Gameweek definitions | — | **writes** |
| Points, leaderboards | — | **writes** |
| Player prices | — | **writes** |
| Migrations / schema | **owns** (Flyway) | proposes + coordinates |

### Consistency & timing you must design for
- **Eventual consistency, ~1-hour granularity.** `football` reflects reality up to
  ~an hour late. Don't design for goal-instant freshness.
- **Data is mutable until a match is final.** Scores change, fixtures get
  postponed, events get corrected/rescinded. **Recompute**, don't cache a result
  as permanent — and even after FT, Sportmonks can issue late corrections, so a
  re-score on a later sync is possible. Make scoring **re-runnable** (compute →
  upsert `points_results`), not append-once.

### Failure independence (why this split is robust)
- **Sync worker down?** Your backend keeps serving the **last-synced** data from
  Postgres — the app stays up, just stale until the sync resumes.
- **Backend down?** The sync worker keeps ingesting; no fantasy data lost, and
  fresh football data is waiting when the API returns.
- **Neither can corrupt the other** — DB grants physically stop the sync worker
  writing `app` and the backend writing `football`/`sync`.

### What each side must never do
- **Backend:** never write `football`/`sync`; never run migrations ad hoc; never
  assume a fixture has two teams or a score.
- **Sync worker:** never writes `app`; never computes fantasy points; PSL-scoped
  ingestion only (extend it before relying on other competitions).

**In one line:** the sync worker fills `football`, your backend turns it into a
game in `app`, and Postgres is the seam between them.
