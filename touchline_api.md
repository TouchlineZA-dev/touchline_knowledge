# Touchline API • football, identity and fantasy

[Team home](MASTER.md) · [Schemas and joins](DB.md) · [UI consumers](touchline_main.md) · [Football producer](touchline_sync.md) · [Sync Sprints 3–5](touchline_sync_sprints_3-5.md)

**Source • `touchline-api`, `main@15b0a53` plus open PRs #14 and #8–#13, 23 November 2026.**
- `main` has Sprint 1 (PR #6, merged).
- Sprints 2–8 are in seven open PRs, all based on `main`. They merge in this order: #14, #8, #9, #10, #11, #12, #13.
- This guide describes the code at the tip of that stack (`sprint/8-sync-alignment`). Where a feature arrives in a later PR, the sprint is named.
- The tip's full suite is 419 passed, 0 skipped, 0 failed, including every PostgreSQL test.
- This replaces the earlier baseline (`phathu-dev@f887905`), which is also on `main`.

The dedicated ASP.NET Core (.NET 10) service is the intended public contract for phones and web. It has no hosted deployment yet. Route paths below are relative to `/api/v1`.

**Status of the rules here:** the rules below are what the code does. The product decisions behind them are **proposed, awaiting team sign-off**. See [MASTER decisions](MASTER.md#decisions-and-open-questions). In particular, they conflict with Phatu's handbook on captaincy, autosubs, budget and Free Hit.

## Trace a request

1. **Startup.** `Program.cs` loads settings and validates them (in Staging and Production only). It registers Npgsql and EF Core, JSON string enums, CORS, rate limiting, authentication and controllers.
2. **Public reads.** A public controller reads Postgres only. The API never calls Sportmonks or the worker: Postgres is the only handoff, and a test enforces it.
   - Every read of `football.match_events` and `football.player_lineups` goes through `Data/FootballReads.cs`. It drops rows where `withdrawn_at` is set (see [withdrawn rows](#withdrawn-rows-and-freshness)).
3. **Protected routes.** These validate the Supabase access token first. The user always comes from the token's `sub`, never from the request body.
4. **Fantasy writes.** These run in one transaction on `app` tables, with a team-row lock and a version concurrency token.
   - The DB role cannot write `football` or `sync`. EF's save guard also refuses writes to them.
5. **Responses.** Public and fantasy GETs carry ETags. Errors are RFC 9110 problem details with a machine-readable `code`.

## Routes (Sprints 1–8)

### Public football and rules

| Method and path | What it returns | Sprint |
|---|---|---|
| GET `/health` | Minimal liveness. No hosts, versions, job names or error text | 1, trimmed in 6 |
| GET `/health/ready` | Database readiness | 6 |
| GET `/health/freshness` | Data freshness. Stale after 48 h; not `fresh` while any live fixture is stalled. Adds `stalledLiveFixtures` (a count) | 6, 8 |
| GET `/seasons/current` | The PSL season flagged current. 503 if none is flagged | 1 |
| GET `/teams` | Clubs in the current PSL season | 1 |
| GET `/clubs`, `/clubs/{id}` | Clubs with stable internal id (used for favourite club) and Sportmonks id. Crest is `null` for a placeholder image. Club detail | 3, 6 |
| GET `/players`, `/players/{id}` | Players with position, current club, price and availability. Opt-in `?page`/`?pageSize`. Player detail by id, never by name | 1, 6 |
| GET `/player-rankings` | Port of the web app's Netlify function | 6 |
| GET `/fixtures?from=…&to=…` | Current-season fixtures. Kickoff field is `kickoffUtc` (`kickoff` is a deprecated alias for the web rollout) | 1 |
| GET `/fixtures/{id}` | Match detail: teams, score, phase, clock, ordered events (paged), lineups, substitutions, team stats, coverage flags, freshness and revision | 1, 8 |
| GET `/matches` | Compatibility shape for the web client | 6 |
| GET `/standings` | PSL table: derived from final results, plus official deductions from `football.standings` | 1 |
| GET `/gameweeks?seasonId=…` | Calendar with persisted week `number` (null before V043) and `readyToScore` | 1, 2 |
| GET `/rules` | The machine-readable rulebook `2026-27.v1`. Every number comes from configuration | 2 |

### Signed-in manager (`Authorization: Bearer <Supabase token>`)

| Method and path | What it does | Sprint |
|---|---|---|
| GET, PATCH `/me` | Find or provision the profile. PATCH sets username, favourite club and onboarding | 3 |
| GET, PATCH `/me/notifications` | Notification preferences | 3 |
| GET `/me/export` | Data export. Needs a sign-in within the last 10 minutes | 3 |
| DELETE `/me` | Account deletion (anonymisation). Needs a recent sign-in. League ownership passes on or the league is archived | 3 |
| GET, POST, PATCH `/fantasy/team` | Read, create or rename the caller's team (name 3–30 characters) | 1, 3 |
| GET `/fantasy/team/snapshot` | Team, squad, chips, free transfers, deadline, latest points and leagues | 1 |
| GET, PUT `/fantasy/team/squad` | Read the squad, or save the first 15 | 1, 4 |
| PUT `/fantasy/team/lineup` | Rearrange the XI and armbands. Send `If-Match` | 1, 4 |
| POST `/fantasy/team/transfers/preview` | A 15-minute quote: costs, bank, free transfers, hit, `valid`, `violations`, `priceRevision` | 4 |
| POST `/fantasy/team/transfers/confirm` | Confirm a quote with an `Idempotency-Key` header. A retry returns the original receipt (`replayed: true`) | 4 |
| POST `/fantasy/team/transfers` | Legacy one-step transfer (with `If-Match`) | 1 |
| GET `/fantasy/team/transfers/ledger` | Transfer history with receipts | 4 |
| POST `/fantasy/team/autopick/preview` | A greedy legal squad suggestion (nothing saved) | 4 |
| GET, POST `/fantasy/team/chips`, DELETE `/fantasy/team/chips/{chip}` | List and play chips. Bench Boost and Triple Captain can be cancelled before the deadline; Wildcard can't | 1, 4 |
| GET `/fantasy/team/points?gameweek=N` | Team total and breakdown, provisional while live | 1, 5 |
| GET `/fantasy/team/points/explain` | Per-player, per-fixture explanation, including correction messages | 5 |
| GET `/fantasy/team/history` | Past gameweeks with rank movement | 5 |
| GET, POST `/fantasy/leagues` | The caller's leagues (including Overall), or create a private league | 1 |
| POST `/fantasy/leagues/join`, `/{id}/leave`, `/{id}/owner`, `/{id}/archive` | Join by exact code, leave, transfer ownership, archive | 1, 3 |
| GET `/fantasy/leagues/{id}/standings`, `/{id}/history` | Standings (`overall` for every team) and history. Rate-limited | 1, 5 |

**Writes need a target gameweek.** Every squad, lineup, transfer and chip write names `gameweekId` (Sprint 4). A write for a locked week never edits the next one.

### Admin (verified admin; support and operators share the role)

| Method and path | What it does | Sprint |
|---|---|---|
| POST `/admin/gameweeks` | Create a week | 1 |
| PUT `/admin/gameweeks/{id}/deadline` | Edit a deadline (only before it passes) | 2 |
| POST `/admin/gameweeks/{id}/lock` | Run the deadline lock (snapshots squads) | 4 |
| PUT `/admin/fixtures/{id}/gameweek`, GET `/admin/fixtures/{id}/gameweek-history` | Move a fixture between weeks with a reason. A locked week needs `confirmLockedChange: true`. Full history | 2 |
| GET, POST `/admin/gameweeks/{id}/settlement` | Settlement state, each fixture's outcome, blockers, `canFinalise`, `amendmentNeeded`. POST records a transition | 5 |
| POST `/admin/fixtures/{id}/resolution` | Record a manual resolution when there's no worker evidence | 5 |
| POST `/admin/gameweeks/{id}/score` | Finalise (publish the final revision atomically) | 1, 5 |
| POST `/admin/gameweeks/{id}/corrections/preview`, `/corrections` | Preview and publish an amendment with source and reason | 5 |
| GET `/admin/gameweeks/{id}/revisions` | Published revisions | 5 |
| GET `/admin/diagnostics` | Alerts, stalled live fixtures, provider quota and freshness | 8 |
| POST `/admin/staging-kit/reset` | Reset staging test accounts. Never enabled in Production | 6 |
| POST, GET `/admin/transfer-windows`; POST `/admin/prices/preview`, `/admin/prices/recalculate` | Transfer windows and price runs | 1 |
| PUT `/admin/players/{id}/price-override`, `/admin/players/{id}/available` | Price and availability overrides with a reason | 1, 2 |
| PUT `/admin/users/{id}/suspension` | Suspend or restore a manager (hidden from Overall; reads only) | 3 |

Admin writes go to `app.admin_audit_log` (V048, Sprint 7). An audit failure never fails the request.

## Rules as built (rulebook `2026-27.v1`, points table `2026-27.points.v1`)

Every number is in `Fantasy/FantasyRules.cs` and served by `GET /rules`.

**Squad and money**
- **Budget:** R100.0m. Money is in millions of rand with one decimal.
- **Squad:** 15 players (2 GK, 5 DEF, 5 MID, 3 ATT), at most 3 from one club.
- **Formations:** 4-3-3, 4-4-2, 3-5-2, 3-4-3, 5-3-2.
- **Deadline:** 90 minutes before the week's first kickoff, shown in UTC and SAST.

**Transfers and chips**
- **Transfers:** 1 free per gameweek, no rollover. Each extra transfer costs −4.
- **Captaincy:** captain ×3, vice-captain ×2.
  - The vice gets the captain's ×3 only if the captain plays **0 minutes across the whole gameweek**.
  - A missing armband is filled by a deterministic pick from the XI.
- **Chips:** each once per season, one per gameweek.
  - Wildcard waives charged hits.
  - Bench Boost counts the bench.
  - Triple Captain: captain ×4, passing to the vice (×4) if the captain plays 0.
- **No Free Hit. No automatic substitutions.**

**Scoring per fixture**

| Event | Points |
|---|---|
| Appearance, 45+ minutes | +1 |
| Full match (started, on the pitch at the final whistle) | +2 in total |
| Goal (event 14) or penalty goal (16) | +3 |
| Assist (the goal's related player) | +2 |
| Clean sheet (GK or DEF, 60+ minutes, team conceded 0 in the match) | +2 |
| Yellow | −1 |
| Red, or second yellow (21) | −2 (a second yellow counts as a red only) |
| Own goal (event 15) | −2 |

**What isn't scored or counted**
- **Not scored:** missed penalties, saves, defensive contributions, bonus points.
- **Unattributed goals and cards:** they count for the team (for example, for clean sheets) but give nobody points.
- **Awarded (17) and walk-over (14):** results stand, with no player points.

**Minutes** come from the XI (lineup type 11), substitutions (event 18) and reds.
- Counting is inclusive: a 46' substitution gives 46 minutes to the player going off and 45 to the player coming on.
- 45+x counts as 45, and 90+x as 90.

**Match states:** all 25 provider states are mapped explicitly (`MatchStateMatrix` in `Business/MatchStatus.cs`). An unrecognised code becomes status `Unknown`, flagged for review and never treated as scheduled.

**Gameweeks:** a fixture belongs to the week it is **assigned** to (`app.fixture_gameweek_assignments`, V043, with history). Double weeks add the scores up. Postponed fixtures leave their week (default decision 1).

**Ties**
- Leagues: points, then overall points, then username, then team id.
- PSL table: points, goal difference, goals for, club name.

## Finalisation and corrections (Sprint 5)

**Before finalising**
- **Settlement is a state machine** per gameweek (V047).
- **Finalising never runs on fixture states alone.** Every finished fixture needs one of two kinds of evidence:
  - the worker's `marked_complete` in `sync.gameweek_sync_status` (0 rows today);
  - or a recorded admin resolution.
- **Unattributed events:** a 48-hour hold applies first, even when worker evidence exists.
- **Suspended fixtures:** not settled, so they block finalising. Awarded and postponed fixtures are settled.

**Finalising**
- `POST /admin/gameweeks/{id}/score` publishes one final revision atomically. An interrupted run resumes on the next call.
- Finalising never rescores entries saved under another rules version.

**After finalising**
- A provider correction changes nothing until an operator previews and publishes an **amendment** with a source and reason.
- An amendment rescores every team and creates a new revision. Managers see a correction message on `/points/explain`.

**Live points:** these are provisional, cached for 60 seconds, and recomputed from the same corrected rows every view reads.

## Withdrawn rows and freshness

**Withdrawn rows** (Sprint 8, with the sync owner's V040/V041)
- The worker marks an event or lineup row the provider stopped listing (a VAR reversal or a reissued correction) with `withdrawn_at`.
- Every API read filters `withdrawn_at IS NULL`. Before V040 the column doesn't exist and every row counts.
- The worker's rescind flag (`rescinded = true`) is also honoured.
- **Deploy order:** the API with this filter (PR #13) must be live before V040 runs in production. V040 is already on sync `main`.

**Revision**
- Match detail shows the worker's revision `s{n}` from `sync.fixture_freshness` when that table has the fixture (`revisionSource` says which).
- Otherwise it shows the API's content hash `r-…`. The hash ignores timestamps and changes on any correction.

**Stale**
- A live fixture is stale when `sync.live_fixture_health.poll_health = 'stalled'`.
- Otherwise freshness uses a 75-minute window. The fallback works before V040.

**"Should be live"** is states {2, 3, 4, 6, 9, 11, 16, 18, 19, 21, 22, 23, 25, 26}, matching the sync owner's V040 view. It drives watching and alerts only. Status and scoring stay as the state matrix says, so suspended, delayed, awaiting-updates and pending fixtures are watched but not scored.

## Error codes

Every error is `application/problem+json` with a `code`; rule violations also list `errors`. Full table: `docs/api/errors.md`.

| Status | `code` | App action |
|---|---|---|
| 400 | `rule_violation`, `insufficient_funds` | Show the errors or shortfall; keep the selection |
| 400 | `target_gameweek_required`, `idempotency_key_required` | Send `gameweekId`; send a UUID `Idempotency-Key` per confirmation |
| 401 | (JWT handler), `invalid_subject` | Refresh the session or sign in again |
| 403 | `forbidden`, `account_suspended`, `account_deleted`, `reauthentication_required` | Hide the action; show the notice; sign out; ask for a fresh sign-in |
| 404 | `not_found` | For example, offer to create a team |
| 409 | `stale_squad`, `gameweek_locked`, `squad_already_picked` | Reload `current`; move the edit to `current.nextGameweek`; load the saved squad |
| 409 | `price_changed`, `quote_expired`, `idempotency_key_reused` | Show the new quote in `current`; use a new key |
| 409 | `username_taken`, `chip_not_cancellable`, `legacy_email_unlinked`, `email_in_use` | Ask again; explain; show the account-recovery text |
| 429 | (rate limit) | Back off and retry |
| 503 | title "Schema not ready", "Sportmonks codes not confirmed" | A migration or code mapping is missing: retry later |

## Migrations this API depends on

API files `docs/migrations/007–012` are canonical in touchline-sync as **V043–V048** (offset 36). Each is safe to rerun and grants no DELETE. The API works before and after each one: a feature whose migration is missing returns `503 Schema not ready`. Status and apply order are in [DB.md](DB.md#v043v048-api-sprints-27).

## Local development

**Setup:** use the .NET 10 SDK, a development PostgreSQL migrated by Flyway, and the matching development Supabase Auth project. Copy `.env.example` to `.env` and fill in `ConnectionStrings__Touchline` and `Supabase__Url` privately. Connect as `app_api_user`.

```sh
dotnet restore Touchline.sln
dotnet run --project src/Touchline.Api
dotnet test Touchline.sln
```

**PostgreSQL tests**
- The tests build a disposable database from the pinned migrations in `tests/db/flyway`: sync `main` at `3d63fd0` plus V042, then V043–V048.
- CI uses Testcontainers with `TOUCHLINE_REQUIRE_DB_TESTS=true`.
- Locally, point `TOUCHLINE_TEST_PG` at any throwaway server.

**Hosting:** see `docs/api/deployment.md` and `docs/ops/release-runbook.md`. The host is still an open decision.

## What's done / what's missing

### Done (Sprints 1–8; PR #6 merged, #14 and #8–#13 open)

| Sprint | Delivered |
|---|---|
| 1 Football facts | One state matrix for all 25 states, the match clock, the PSL table, match detail, the revision, and Postgres-only reads |
| 2 Season rules | Rulebook and `GET /rules`, persisted week numbers and fixture assignment (V043), rules versions (V044), reference week (65 / 61) |
| 3 Accounts | Usernames, favourite club, export, deletion, suspension, notification preferences (V045) |
| 4 Safe writes | Target gameweek, row lock and version, quotes and idempotent receipts, deadline lock snapshot, chip cancellation (V046) |
| 5 Settlement | Settlement state machine, resolutions, revisions and amendments, explain and history (V047) |
| 6 Frontend contract | Clubs, player and club detail, rankings, compatibility routes, paging, OpenAPI, examples, staging kit |
| 7 Operate and secure | Health split, rate limits, startup validation, admin audit log (V048), CI secret scan, access matrix |
| 8 Sync alignment | `withdrawn_at` filter, worker freshness and revision, stalled-fixture alerts, admin diagnostics, migrations renumbered to V043–V048 |

Phatu's checklist coverage is 97 API-owned items: 89 done, 5 partial (blocked on other owners or staging), 3 other owner or conflict. The detail is in `docs/sprints/PHATU-COVERAGE.md`.

### Sprint 9 (planned, not built)

- **Read the sync owner's V041 tables in the API:**
  - `sync.fixture_completeness`, as worker evidence for settlement next to `gameweek_sync_status`, replacing most manual resolutions;
  - `sync.late_corrections`, to raise `amendmentNeeded` and the correction message;
  - `sync.fixture_schedule_changes`, to drive fixture reassignment;
  - `sync.event_changes`, to explain corrections on `/points/explain`.
- **Remove Google from the provider checklist** (`docs/accounts/provider-checklist.md`). The agreed sign-in options are Facebook and Apple ([AUTH](AUTH.md)).

### Missing: needs the sync team

1. **Rename the sync owner's V042 to V049.** V043 merged to sync `main` before V042. Flyway won't apply an older pending version after a newer one, unless `outOfOrder` is used. Don't run `flyway migrate` on a shared database until this is resolved.
2. **Restart the data feed:**
   - fix the event and lineup sync, which stopped on 1 Nov;
   - backfill the 2025/26 lineups.
3. **One live writer:** cut over from the Netlify football schedules to the Python worker at one recorded time, or have both writers take V042's `scheduler_lease`.
4. **Populate `sync.gameweek_sync_status`** (0 rows). Until then, every finished fixture needs an admin resolution before finalising.
5. **Exclude postponed fixtures** (state 10) from completeness (`assess_fixture()`).
6. **Use the API's gameweek assignment:** once V043 is applied, `sync.gameweek_for()` and the reschedule triggers should use `app.fixture_gameweek_assignments` (newest row per fixture). That needs `SELECT` on it for the worker.
7. **Reissued corrections:** mark the stale event withdrawn or rescinded, or delete it. The API handles all three.
8. **Credentials:**
   - rotate the `sync_worker_user` password, which is committed in V001;
   - remove passwords from V001.
9. **Repo hygiene:** fix touchline-sync's 14 Dependabot alerts (4 critical) and run a full-history secret scan.
10. **Guide fixes:**
    - BACKEND_INTEGRATION.md: budget 100.0, price default 4.0, all 25 states, the gate exists;
    - own goal is event 15;
    - short codes for 5 clubs.
11. **Provider timestamp:** store Sportmonks' own update time, so provider delay can be measured.

### Missing: needs other owners

1. **Sportmonks account holder:** the key is on the free plan, with no PSL data since 18 Nov. Restore the subscription. The API keeps serving the last data with stale flags.
2. **Ops:**
   - rotate `app_api_user` (its password is also in V001);
   - staging load test (`docs/load`);
   - timed restore;
   - rotation rehearsal;
   - production-role access tests;
   - run the deadline lock and settlement routes each gameweek, or automate them;
   - log retention.
3. **Supabase dashboard owner:**
   - provider credentials and callbacks (`docs/accounts/provider-checklist.md`);
   - an Apple secret rotation owner;
   - Facebook live;
   - MFA for admins;
   - two staging kit accounts.
4. **Mobile owner:**
   - callback round-trip sign-off;
   - the staging kit flow;
   - a push provider (`docs/api/delivery-proposal.md`).
5. **Web owner** (`docs/api/web-migration.md`):
   - switch to `kickoffUtc`;
   - retire synthetic ratings;
   - send `gameweekId` and idempotency keys;
   - the club slug map;
   - stop storing passwords in localStorage.
6. **News-service owner:** decide on `docs/api/news-proposal.md`.
7. **Support:** `legacy_email_unlinked` linking, Supabase Auth deletion within 30 days, and how to use `/points/explain`.
8. **Legal and product:**
   - image hotlinking and retention rights;
   - provider-derived fixtures stored in git;
   - rulebook sign-off.
9. **Privacy officer:** POPIA review (`docs/security/privacy-inventory.md`).
10. **Independent security reviewer:** review and retest.

## File catalogue

The tip of the stack has **224 tracked files**. Paths start at the checkout. Source files are listed one by one. Tests, docs and SQL are listed by folder.

### Repository root and CI

| File | Responsibility |
|---|---|
| `.editorconfig`, `Directory.Build.props`, `Touchline.sln` | Formatting, shared MSBuild properties, solution |
| `.env.example` | Setting names only. Real values stay private per environment |
| `.gitleaks.toml` | Secret-scan configuration |
| `.github/workflows/ci.yml` | Release build, full tests with required PostgreSQL tests (including the OpenAPI contract test), dependency vulnerability check, gitleaks scan of commits since 23 Nov |
| `README.md` | Setup and orientation |

### `src/Touchline.Api`

| File | Responsibility |
|---|---|
| `Program.cs` | Composition: DB, services, auth, CORS, rate limits, controllers |
| `DotEnv.cs` | Loads the nearest `.env`; real environment variables win |
| `OpenApiSetup.cs` | OpenAPI document with shared problem-details responses (exported to `docs/api/openapi-v1.json`) |
| `Operations.cs` | Incident switches and rate limits (`docs/ops/release-runbook.md`) |
| `appsettings.json`, `appsettings.Development.json`, `Properties/launchSettings.json`, `Touchline.Api.csproj` | Configuration and project |

### `src/Touchline.Api/Business`

| File | Responsibility |
|---|---|
| `MatchStatus.cs` | The state matrix (25 states → status and phase, `Unknown` for others) and the "should be live" set |
| `MatchClock.cs` | Where a match minute comes from (no live clock on the provider plan) |
| `MatchFactsDecoder.cs` | Pure decoding of lineups and events into one player's facts per fixture |
| `Position.cs` | Position ids 24–27 → GK, DEF, MID, ATT |
| `SportmonksCodes.cs` | Provider codes: states, events (14, 15, 16, 18, 19, 21), lineup types |

### `src/Touchline.Api/Controllers`

| File | Responsibility |
|---|---|
| `ClubsController.cs`, `TeamsController.cs` | Clubs in the current season, club detail |
| `CompatibilityControllers.cs` | `/player-rankings` and `/matches` for the web client |
| `FantasyControllers.cs` | Fantasy team, leagues and admin routes. The user always comes from the token |
| `FantasyFilters.cs` | Service exceptions → problem details. ETags |
| `FixturesController.cs` | Fixture list and match detail |
| `GameweeksController.cs`, `SeasonsController.cs` | Calendar and current season |
| `HealthController.cs` | `/health`, `/health/ready`, `/health/freshness` |
| `MeController.cs` | Account, notifications, export, deletion |
| `PlayersController.cs` | Player list and detail |
| `RulesController.cs` | The rulebook |
| `StandingsController.cs` | PSL table |
| `TouchlineControllerBase.cs` | Opt-in paging for public lists |

### `src/Touchline.Api/Data`

| File | Responsibility |
|---|---|
| `FootballReads.cs` | The only read path for events and lineups. Filters `withdrawn_at` |
| `SchemaFeatures.cs` | Detects which optional tables and columns exist, so features degrade before a migration |
| `TouchlineDbContext.cs` | EF mapping over `football` and `sync` (read-only) and `app` |

### `src/Touchline.Api/Fantasy`

| File | Responsibility |
|---|---|
| `FantasyRules.cs` | Every rule number, bound from configuration |
| `Rulebook.cs` | The versioned rulebook served at `/rules` |
| `FantasyModels.cs`, `SquadRules.cs`, `TransferAndChipRules.cs` | Formations, squad and lineup validation, transfers and chips |
| `Scoring.cs` | Player facts → points; armbands, chips, hits |
| `GameweekRules.cs` | Fixture membership, deadlines, finalising rules |
| `PlayerValuation.cs` | Price model from past seasons |
| `README.md` | The fantasy module's notes |

### `src/Touchline.Api/Models`

| File | Responsibility |
|---|---|
| `AppEntities.cs`, `FantasyEntities.cs` | `app` entities: users, gameweeks, teams, entries, receipts, settlement |
| `FootballEntities.cs`, `SyncEntities.cs` | Read models for `football` and `sync` |
| `FantasyContracts.cs`, `ResponseModels.cs`, `MatchDetailModels.cs` | Request and response shapes |

### `src/Touchline.Api/Services`

| File | Responsibility |
|---|---|
| `FootballService.cs`, `FootballDetailService.cs` | Seasons, players, fixtures, prices; player and club detail, rankings |
| `MatchDetailService.cs` | Match detail, freshness, revision (`s{n}` or content hash) |
| `MatchFactsService.cs`, `PlayerIdBridge.cs` | Load events and lineups and map Sportmonks ids to internal players |
| `StandingsService.cs` | PSL table with official deductions |
| `HealthService.cs` | Readiness, freshness, stalled live fixtures |
| `GameweekService.cs` | Calendar, assignments, deadlines |
| `FantasyContextService.cs`, `FantasyTeamService.cs`, `SquadStore.cs` | Caller, season, team, writable week; team actions; squad persistence |
| `SafeWriteService.cs` | Transfer quotes, confirmations, receipts, idempotency |
| `AvailabilityService.cs` | Who can be picked and priced |
| `PointsService.cs` | Provisional and final points |
| `SettlementService.cs` | Settlement state, resolutions, revisions, amendments |
| `LeagueService.cs` | Private leagues and Overall |
| `PricingService.cs` | Price runs, windows and overrides |
| `StagingKitService.cs` | Staging test accounts (never in Production) |
| `FootballOptions.cs`, `ServiceExceptions.cs` | Options; typed domain failures |

### `src/Touchline.Api/Users`

| File | Responsibility |
|---|---|
| `SupabaseAuthSetup.cs`, `SupabaseAuthOptions.cs`, `SupabaseSigningKeys.cs` | Token validation; JWKS cache and refresh |
| `CurrentUser.cs` | Subject and optional email from the token |
| `UserService.cs` | Links a Supabase user to `app.users` |
| `AccountService.cs`, `AccountOptions.cs`, `UsernameRules.cs` | Account features; username rules (3–20 characters, unique ignoring case) |

### Tests (`tests/`)

| Folder | What it covers |
|---|---|
| `Touchline.Api.Tests/Football` | State matrix, match detail, standings, architecture (no outbound calls), sync alignment |
| `Touchline.Api.Tests/Fantasy` | Squad, scoring, transfers, chips, leagues, valuation |
| `Touchline.Api.Tests/Reference` | Reference week (65 / 61) and the fixture replay |
| `Touchline.Api.Tests/Accounts`, `Auth` | Accounts, token validation, `/me` |
| `Touchline.Api.Tests/SafeWrites`, `Settlement`, `Security`, `Contract` | Concurrency and idempotency, settlement, access, OpenAPI contract |
| `Touchline.Api.Tests/Db` | Clean install and populated upgrade of every migration |
| `db/flyway` | Pinned canonical migrations for the test database |
| `db/reference`, `db/replay` | Reference-week snapshot and a recorded fixture replay |

### Docs (`docs/`)

| Folder | Contents |
|---|---|
| `api/` | Errors, examples, OpenAPI, deployment, staging kit, web migration, delivery and news proposals |
| `rules/` | Rulebook `2026-27.v1`, coverage matrix, reference week |
| `accounts/` | Account policies, provider checklist, club mapping |
| `migrations/` | API copies 001–012; 007–012 are canonical in touchline-sync as V043–V048 |
| `ops/` | Fantasy operations, matchday runbook, release runbook |
| `security/` | Access matrix, effective-permissions check, privacy inventory |
| `load/`, `release/`, `audit/`, `plans/` | Load test, release record, fantasy audit, plans |
| `sprints/` | Runbook, rules, the 8 sprint reports, PROGRESS, FINAL-SUMMARY, Phatu coverage |
