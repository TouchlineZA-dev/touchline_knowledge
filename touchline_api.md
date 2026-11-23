# Touchline API • football, identity and fantasy

[Team home](MASTER.md) · [Schemas and joins](DB.md) · [UI consumers](touchline_main.md) · [Football producer](touchline_sync.md)

**Source • `touchline_api`, `phathu-dev@f887905`, 23 November 2026.** The dedicated ASP.NET Core service is the intended public contract for phones and web. Phatu reports it has no hosted deployment yet. The similarly named folder inside Main implements a different contract. Route paths below are relative to `/api/v1`.

## Trace a request

1. `Program.cs` loads development settings, requires the Touchline connection string, registers Npgsql and the EF Core context, configures JSON string enums, CORS, authentication, authorization and controllers. Reads default to no tracking. A writer explicitly tracks or adds changed app entities.
2. A public controller resolves its service. For example, `FixturesController` resolves the current season for Sportmonks league 806, accepts optional ISO date bounds and calls `FootballService`. The service converts date bounds to UTC, filters on **internal** season ID, orders by kickoff, joins club names and projects DTOs. The response contains `season`, `count` and `fixtures`. Unknown opponents can read “TBD”. A missing score is meaningful before kickoff.
3. Protected routes validate the Supabase access token first. The issuer is the environment’s Auth URL, the audience defaults to `authenticated`, and the configured signing keys verify the signature. `CurrentUser` takes the UUID from the token subject. `UserService` resolves it to `app.users.id`. Someone supplying another user’s ID in a request cannot use that field to choose which team to edit.
4. A fantasy controller loads the user, season, team, gameweek calendar and applicable rules. A rule violation returns structured errors. A valid change goes through the appropriate service and transaction to `app`. EF’s save guard also rejects writes to entities mapped to `football` and `sync`. The API reads committed worker output between requests.
5. The response contains a projection for the screen. A squad or snapshot GET carries an ETag with the version. The client can reload that representation after a change. Development also exposes the route contract at `/openapi/v1.json` and Scalar at `/scalar`.

## Contract by user journey

| Method and path | Who and what |
|---|---|
| GET `/health` | Public. Queries the latest sync job and reports DB reachability. Returns 503 if the query fails. A reachable DB can contain stale football facts. |
| GET `/seasons/current` | Public. Resolves the PSL season flagged current. Ensure only one such season for predictable selection. |
| GET `/teams` | Public. Lists non-placeholder teams in name order, including crest where available. This query has no season filter. |
| GET `/players` | Public. Lists players with decoded position, latest lineup club for the current season, image and season price. A missing price falls back to the position floor. The player collection itself is not restricted to those with a current-season lineup. |
| GET `/fixtures?from=…&to=…` | Public. Current season, inclusive time bounds, fixture IDs, names, score, raw state and decoded status. Kickoff field is `kickoffUtc`. |
| GET `/gameweeks?seasonId=…` | Public. Returns calendar dates, season and `readyToScore` derived from marked sync rows. |
| GET `/me` | Bearer. Finds or provisions one app profile per Supabase subject. The DB keeps the integer profile ID across identity linking. |
| GET, POST `/fantasy/team` | Bearer. Read or create the caller’s team for this season. POST takes `{"name":"Soweto XI"}`. A duplicate conflicts. |
| GET `/fantasy/team/snapshot` | Bearer. Team, squad, chips used, free transfers, deadline, latest points, leagues and generation time. Latest points can be absent. A client must implement persistent caching for offline viewing. |
| GET, PUT `/fantasy/team/squad` | Bearer. GET accepts an optional gameweek **number**. PUT saves the first complete pick with formation, internal player IDs for starters and ordered bench, captain, vice and optional intended week. Repeating initial selection after a squad exists conflicts. |
| PUT `/fantasy/team/lineup` | Bearer. Rearrange existing players for the writable week and set captaincy. Send the squad ETag in `If-Match`. |
| POST `/fantasy/team/transfers` | Bearer. Send equal-length `out` and `in` internal player-ID arrays and optional gameweek number. Validates squad and funds, records swaps, updates the balance and returns the cost and squad. |
| POST `/fantasy/team/chips` | Bearer. Takes `chip` and optional gameweek. Records seasonal use and the entry’s active chip. |
| GET `/fantasy/team/points?gameweek=N` | Bearer. Defaults to the latest week whose deadline has passed. Returns team total, breakdown and provisional/fixture information where available. |
| GET, POST `/fantasy/leagues` | Bearer. List the caller’s leagues including Overall, or create a named private league. The creator’s team joins immediately. |
| POST `/fantasy/leagues/join` and `/fantasy/leagues/{id}/leave` | Bearer. Join by exact code, leave by numeric league ID. The owner cannot leave. Leaving sets a timestamp and rejoining clears it. |
| GET `/fantasy/leagues/{id}/standings?gameweek=N` | Bearer. Private league membership matters. Use `overall` for all teams in the season. Optional week selects gameweek standings. |
| POST `/admin/gameweeks` | Verified admin. Internal season ID, name, date bounds and optional explicit UTC deadline. Validates overlaps. |
| POST `/admin/gameweeks/{id}/score` | Admin. Uses the **database gameweek ID**. Recomputes the breakdowns and team totals in a transaction. |
| POST, GET `/admin/transfer-windows` | Admin. Create official season windows with start/end dates or list them by season. |
| POST `/admin/prices/preview?seasonId=N` and `/admin/prices/recalculate?windowId=N` | Admin. Preview valuation without saving, or apply a window run. A repeated run requires an explicit force choice. |
| PUT `/admin/players/{id}/price-override` | Admin. Internal player ID, season, expected minutes and/or price, plus a reason. The schema’s availability override table is a separate feature with no corresponding controller here. |

## Example handoff from mobile to the API

After signing in, call `GET /api/v1/me` with the Supabase access token, then `POST /api/v1/fantasy/team` with a name. Read the current season’s player list and the team’s next gameweek. The first selection uses `PUT /api/v1/fantasy/team/squad`. This example illustrates the shape. Replace each example ID with an eligible internal player ID and the week number returned for the target season.

```json
{
  "formation": "4-4-2",
  "starters": [101, 201, 202, 203, 204, 301, 302, 303, 304, 401, 402],
  "bench": [102, 205, 305, 403],
  "captainId": 401,
  "viceCaptainId": 301,
  "gameweek": 10
}
```

In this illustration the 100s are keepers, 200s defenders, 300s midfielders and 400s attackers. Real IDs carry no positional meaning. For an accepted save, compare the returned team ID, 15 players, version and remaining budget with a fresh snapshot on the second device. To change only the captain, retrieve the current ETag and send the same player selection with revised armbands to the lineup route. To replace a midfielder, send a transfer payload such as `{"out":[301],"in":[399],"gameweek":10}` to the transfers route. The API checks the incoming player's actual position, price and club.

## Identity and response handling

The API can verify tokens from the project’s JWKS endpoint and supports an explicitly configured legacy JWT secret. Google/Apple provider credentials belong in the Auth provider configuration. Being signed in to a provider still requires a Supabase session from the correct project. Native social callbacks remain client work.

| Response | Client or operator action |
|---|---|
| 400 validation | Display each item in `errors`. Keep the selection so the fan can fix it. An unreadable version header is also invalid. |
| 401 / 403 | Restore or renew the session for 401. A signed-in fan without membership or admin authority cannot perform the restricted operation. |
| 404 | An absent team invites creation. An unknown week requires calendar handling. The public current-season lookup can return 503 when no PSL season is flagged. Use the originating route to choose the state. |
| 409 account conflict | On `/me`, `legacy_email_unlinked` calls for verified support-assisted linking. `email_in_use` calls for sign-in via the original identity and provider linking. Email alone cannot establish ownership. |
| 409 squad conflict | Use the returned `current` state when present, reload the squad and let the fan review the edit. Handle locked weeks and duplicate creates separately. |
| 304 | A matching `If-None-Match` GET has no response body. Reuse the corresponding account-scoped representation. |
| 503 | Inspect the problem title and logs. Missing schema, unresolved minute codes and database unavailability require different fixes. The older migration numbers in an error message should be reconciled with the [Flyway guide](DB.md). |

`If-Match` is accepted by lineup and transfer writes. Prove two simultaneous writes to the same version on PostgreSQL before promising lost-update protection. Browser clients also need ETag exposed by CORS to read it across origins. A UI retry after a network timeout should reload server state before repeating a purchase or transfer.

## What constitutes a legal team

Money is in **millions of rand**. A value of `7.5` displays as R7.5m. `FantasyTeamService` explicitly assigns the configured R300.0m when creating a team. An initial squad has 2 goalkeepers, 5 defenders, 5 midfielders and 3 attackers. No duplicates, at most three from one real club, and no unknown playing position. A missing club can produce a warning. Position IDs 24–27 decode to GK, DEF, MID and ATT respectively.

The XI has one keeper and a permitted DEF–MID–ATT formation: 4-3-3, 4-4-2, 3-5-2, 3-4-3 or 5-3-2. Four players sit on the ordered bench, including one keeper. Captain and vice are separate starters. A new week can show the previous picks before materialising that week’s rows. At the deadline, the service can fill missing armbands from the XI. Once a fan has picked the initial 15, use lineup changes to rearrange the same players and transfers to change personnel.

One free transfer per gameweek, no rollover, then four points per additional transfer. The three chips are `wildcard`, `bench_boost` and `triple_captain`. Each is usable once per team per season, with one chip in a week. A wildcard removes transfer hits, Bench Boost counts the bench, and Triple Captain changes the armband multiplier. No automatic bench substitutions are implemented.

## How a match becomes a score

A kickoff belongs to a gameweek if its **SAST calendar date** is within that season’s inclusive dates. The stored deadline wins, otherwise the service derives it from the first kickoff. Calendar order supplies the week number. Keep week number, week row ID, provider season ID and internal season ID distinct.

`MatchFactsService` joins lineup rows and events to internal players through Sportmonks IDs. Starter/bench status and substitution direction determine minutes. `Scoring.cs` accumulates per-fixture contributions. A 45-minute appearance earns 1 point, a full match 2 in total. A goal earns 3, an enabled assist 2, a qualifying clean sheet for a GK/DEF 2 after at least 60 minutes. Yellow costs 1, red costs 2 and an own goal costs 2. A captain who plays has a ×3 multiplier and a playing vice normally has ×2. Triple Captain raises the armband to **×9 in code**. If the captain plays zero minutes the vice takes the armband. Confirm this in release copy because descriptive text in the fantasy README also mentions ×4.

**Worked example under these settings.** A starting midfielder plays the full match and scores one goal, with no other scoring events. The base contribution is 2 + 3 = 5. As captain that is 15, or 45 under Triple Captain. Transfer hits are applied to the team total after counted player contributions. `points_results.points` stores the base score. The breakdown and team entry explain why the manager’s total differs.

A provisional computation can be cached for 60 seconds. Finalisation accepts a marked-complete sync row **or** a nonempty set of entirely final fixtures (state IDs 5, 7, 8). It recomputes and upserts each team/player/week breakdown, saves each team total and clears the week cache. A final whistle alone can precede the full event feed. Agree the evidence needed for publishing official scores and which operator reruns a corrected week. Starter, bench and substitution direction are still unset in `Business/SportmonksCodes.cs`. The protected scoring and price paths can return 503. A missing points value should remain distinguishable from a genuine zero.

Private leagues support 20 teams by default. Membership and owner checks constrain access. Standings are derived from team totals with gameweek/overall points and username tie-breaking. Overall is a virtual season-wide league. For price changes, `PricingService` gathers PSL-only history, `PlayerValuation` smooths points per 90 toward a position average, weights past seasons 0.6/0.3/0.1, projects minutes, ranks within position and rounds to R0.5m. A new player can fall back to the position floor. Preview before a transfer-window run, record overrides with a reason and preserve price history.

## Local development and the first hosted release

From the dedicated checkout, use the .NET 10 SDK, a development PostgreSQL database migrated by Sync, and the matching development Supabase Auth project. Populate a local `.env` from `.env.example` with `ConnectionStrings__Touchline` and `Supabase__Url`. Deployment environment variables take precedence. Configure `Supabase__JwtSecret` only for a project using legacy symmetric signing. Use `app_api_user` for SQL.

```sh
dotnet restore Touchline.sln
dotnet run --project src/Touchline.Api
# Separate terminal, from the repo root
dotnet test Touchline.sln
```

The development HTTP profile is on port 5147. Open `/scalar`, then compare health, current season, fixtures and a signed-in `/me` request. Auth integration tests need Docker for disposable PostgreSQL. The checked-in fixture still reads a missing `docs/proposed-migrations/V033__app_users_supabase_auth_link.sql`. Bring the canonical V033 from Sync into the test build through a pinned migration input. Skipped DB tests cannot demonstrate a successful migration.

**Hosting recommendation for discussion:** deploy the dedicated service to Azure App Service using a supported .NET 10 runtime or a container you build with that runtime. Keep Netlify web and ingestion operational. Use a team-owned HTTPS hostname and separate staging/production settings. Configure `ASPNETCORE_ENVIRONMENT=Production`, a runtime connection string, the matching Auth URL, explicit `Cors__AllowedOrigins__0` etc., and the host’s listening port. Use `dotnet publish src/Touchline.Api -c Release` for a framework-dependent release when the host supplies the runtime. Register `/api/v1/health` as the readiness probe. Store credentials in hosting settings and keep normal API startup independent of schema migration. [Microsoft hosting configuration](https://learn.microsoft.com/en-us/azure/app-service/configure-language-dotnetcore)

First migrate staging, deploy the API, check DB and auth, then install an Expo preview build pointing at it. Confirm 401 without a token, 200 with the right user, denial for another user’s private league, recovery from a stale edit, and a clean signed-in cold start. Keep the previous service revision available for rollback and keep schemas backward-compatible with installed versions. Record URLs and deploy IDs in the [master decision record](MASTER.md) when verified. The main gaps before a single public base URL are standings, rankings, match detail and news routes with shapes both clients can consume.

## File catalogue

This revision contains **73 tracked files**. Combine each directory heading with the filename beneath it. The entries name responsibilities and selected declarations or direct source dependencies. Installed libraries, Git metadata, caches and intermediate builds are local machine state. The private `.env` supplies runtime settings. Share setting names through the example files.

### Repository root

| File | Responsibility and connections |
|---|---|
| `.editorconfig` | Maintains C# formatting and analysis conventions in IDEs. |
| `.env.example` | Lists environment setting names and formats. Supply actual values privately per environment. |
| `.gitignore` | Git exclusions for local settings and generated files. |
| `Directory.Build.props` | Applies shared MSBuild properties to the solution. |
| `README.md` | Repository reference for . setup and use. Compare its assumptions with the inspected code. |
| `Touchline.sln` | Groups the API and its test project for restore, build and CI. |

### `.github/workflows`

| File | Responsibility and connections |
|---|---|
| `ci.yml` | Builds Release on .NET 10 and runs the solution tests with required Docker-backed auth coverage. |

### `docs/proposed-migrations`

| File | Responsibility and connections |
|---|---|
| `V034__fantasy_core.sql` | Historical proposal for fantasy state. Use Sync’s numbered V035 for the canonical migration. |

### `docs`

| File | Responsibility and connections |
|---|---|
| `staging-smoke-test-social-sign-in.md` | Repository reference for staging smoke test social sign in. Compare its assumptions with the inspected code. |

### `src/Touchline.Api/Business`

| File | Responsibility and connections |
|---|---|
| `MatchStatus.cs` | Decodes provider fixture states for mobile responses and determines finality for scoring. Defines `MatchStatus`, `MatchStatusExtensions`. |
| `Position.cs` | Maps provider position IDs to the four fantasy roles or Unknown. Defines `Position`, `PositionExtensions`. |
| `SportmonksCodes.cs` | Central provider codes including the unset lineup/substitution mappings that gate minutes. Defines `SportmonksCodes`, `PositionIds`, `EventTypes`. |

### `src/Touchline.Api/Controllers`

| File | Responsibility and connections |
|---|---|
| `FantasyControllers.cs` | Routes protected team, league and admin operations and passes token-derived users into services. Defines `FantasyControllerBase`, `FantasyTeamController`, `FantasyLeaguesController`. |
| `FantasyFilters.cs` | Converts domain failures to ProblemDetails and adds ETags and private cache semantics. Defines `FantasyExceptionFilter`, `ETagAttribute`. |
| `FixturesController.cs` | HTTP boundary for fixtures operations. Dispatches to the service layer. Defines `FixturesController`. |
| `GameweeksController.cs` | HTTP boundary for gameweeks operations. Dispatches to the service layer. Defines `GameweeksController`. |
| `HealthController.cs` | HTTP boundary for health operations. Dispatches to the service layer. Defines `HealthController`. |
| `MeController.cs` | HTTP boundary for me operations. Dispatches to the service layer. Defines `MeController`. |
| `PlayersController.cs` | HTTP boundary for players operations. Dispatches to the service layer. Defines `PlayersController`. |
| `SeasonsController.cs` | HTTP boundary for seasons operations. Dispatches to the service layer. Defines `SeasonsController`. |
| `TeamsController.cs` | HTTP boundary for teams operations. Dispatches to the service layer. Defines `TeamsController`. |
| `TouchlineControllerBase.cs` | Creates a common 503 response when no current PSL season is flagged. Defines `TouchlineControllerBase`. |

### `src/Touchline.Api/Data`

| File | Responsibility and connections |
|---|---|
| `TouchlineDbContext.cs` | Maps PostgreSQL columns across three schemas and guards writes to upstream-owned facts. Defines `TouchlineDbContext`. |

### `src/Touchline.Api`

| File | Responsibility and connections |
|---|---|
| `DotEnv.cs` | Loads nearest development environment file while retaining real environment precedence. Defines `DotEnv`. |
| `Program.cs` | Composes database, business services, authentication, CORS and the controller middleware pipeline. Defines `Program`. |
| `Touchline.Api.csproj` | MSBuild framework and NuGet dependency settings for Touchline.Api. |
| `appsettings.Development.json` | ASP.NET development overrides loaded at startup. |
| `appsettings.json` | ASP.NET base configuration loaded at startup. |

### `src/Touchline.Api/Fantasy`

| File | Responsibility and connections |
|---|---|
| `FantasyModels.cs` | Value types for formations, selections, chips and validation used by the pure rules. Defines `Formation`, `FantasyPlayer`, `SquadSelection`. |
| `FantasyRules.cs` | Binds configurable gameplay and pricing constants used throughout the domain. Defines `FantasyRules`, `PricingRules`, `PriceRange`. |
| `GameweekRules.cs` | Calculates SAST date membership, deadlines, finality, league and version-check behaviour. Defines `GameweekRules`, `LeagueRules`, `StandingInput`. |
| `PlayerValuation.cs` | Computes expected return, position-relative prices and backtest metrics from history. Defines `SeasonRecord`, `PriceOverride`, `ValuationInput`. |
| `README.md` | Repository reference for src/Touchline.Api/Fantasy setup and use. Compare its assumptions with the inspected code. |
| `Scoring.cs` | Converts minutes and events into base points and applies lineup, armband, chip and transfer effects. Defines `PlayerFixtureFacts`, `PointsBreakdown`, `PlayerScoring`. |
| `SquadRules.cs` | Validates roster structure and applies legal changes subject to price and role constraints. Defines `SquadRules`, `TransferOutcome`. |
| `TransferAndChipRules.cs` | Calculates transfer costs and checks chip eligibility. Defines `TransferRules`, `TransferCost`, `ChipUse`. |

### `src/Touchline.Api/Models`

| File | Responsibility and connections |
|---|---|
| `AppEntities.cs` | EF entities for application-owned data used by profile, calendar and fantasy operations. Defines `AppUser`, `Gameweek`, `FantasyTeam`. |
| `FantasyContracts.cs` | Typed HTTP request and response shapes shared by fantasy controllers and services. Defines `CreateTeamRequest`, `SquadRequest`, `TransfersRequest`. |
| `FantasyEntities.cs` | EF entities for per-week entries, transfers, chips, memberships and pricing administration. Defines `FantasyGameweekEntry`, `FantasyTransfer`, `FantasyChipUsage`. |
| `FootballEntities.cs` | Read models for ingested provider facts referenced by football and scoring services. Defines `Competition`, `Season`, `Team`. |
| `ResponseModels.cs` | Public response records for football, calendar and health reads. Defines `SeasonSummary`, `TeamSummary`, `PlayerSummary`. |
| `SyncEntities.cs` | Read models for worker job and gameweek readiness information. Defines `SyncJob`, `SyncLog`, `GameweekSyncStatus`. |

### `src/Touchline.Api/Properties`

| File | Responsibility and connections |
|---|---|
| `launchSettings.json` | Configures local HTTP/HTTPS launch profiles and opens Scalar in Development. |

### `src/Touchline.Api/Services`

| File | Responsibility and connections |
|---|---|
| `FantasyContextService.cs` | Resolves user, current season, team and writable week and checks client versions. Defines `FantasyContext`, `FantasyContextService`. |
| `FantasyTeamService.cs` | Coordinates team creation, initial selection, rearrangements, transfers, chips and snapshot assembly. Defines `FantasyTeamService`. |
| `FootballOptions.cs` | Declares the binding for the target PSL provider competition. Defines `FootballOptions`. |
| `FootballService.cs` | Projects season, players, fixtures, current clubs and prices for API reads. Defines `FootballService`, `PlayerClubRow`. |
| `GameweekService.cs` | Orders season weeks, finds matching fixtures, resolves deadlines, reads readiness and creates weeks. Defines `GameweekInfo`, `SeasonCalendar`, `GameweekService`. |
| `HealthService.cs` | Tests DB reachability through the job journal and returns its latest record. Defines `HealthService`. |
| `LeagueService.cs` | Creates leagues, checks memberships and capacity, and assembles private and overall rankings. Defines `LeagueService`. |
| `MatchFactsService.cs` | Resolves provider events and lineups to player facts and requires confirmed minutes mappings. Defines `PlayerFacts`, `MatchFactsService`. |
| `PointsService.cs` | Builds provisional totals and finalises per-player breakdowns and team scores transactionally. Defines `GameweekPlayerPoints`, `TeamTotals`, `PointsService`. |
| `PricingService.cs` | Loads historical evidence, previews or applies a window valuation, and manages admin inputs. Defines `PricingService`, `AdminService`. |
| `ServiceExceptions.cs` | Typed domain failures distinguish missing records, conflicts, denied access and unavailable schema/data. Defines `SchemaNotReadyException`, `UserConflictException`, `RuleViolationException`. |
| `SquadStore.cs` | Loads season prices and clubs, reads or materialises picks carried forward and persists weekly selections. Defines `FantasyDataService`, `LoadedSquad`, `SquadStore`. |

### `src/Touchline.Api/Users`

| File | Responsibility and connections |
|---|---|
| `CurrentUser.cs` | Extracts the authenticated subject and email claims for downstream profile resolution. Defines `CurrentUser`. |
| `SupabaseAuthOptions.cs` | Declares issuer/project, audience and optional legacy signing configuration. Defines `SupabaseAuthOptions`. |
| `SupabaseAuthSetup.cs` | Configures JWT bearer validation and registers signing-key resolution. Defines `SupabaseAuthSetup`. |
| `SupabaseSigningKeys.cs` | Caches public keys and throttles refreshes including after a key rotation. Defines `SupabaseSigningKeys`. |
| `UserService.cs` | Resolves or creates a profile by subject and detects existing-email conflicts. Defines `UserService`. |

### `tests/Touchline.Api.Tests/Auth`

| File | Responsibility and connections |
|---|---|
| `MeEndpointTests.cs` | Regression coverage for Me Endpoint. |
| `PostgresFixture.cs` | Creates disposable ready, legacy and partial-schema DBs for profile integration tests. Defines `PostgresFixture`. |
| `TestApiFactory.cs` | Boots an in-process ASP.NET host with test DB and authentication settings. Defines `TestApiFactory`. |
| `TestTokens.cs` | Builds test-signed identities to exercise the actual bearer validation pipeline. Defines `TestTokens`, `StubJwksHandler`. |
| `TokenValidationTests.cs` | Regression coverage for Token Validation. |

### `tests/Touchline.Api.Tests/Fantasy`

| File | Responsibility and connections |
|---|---|
| `ConflictTests.cs` | Regression coverage for Conflict. |
| `PlayerValuationTests.cs` | Regression coverage for Player Valuation. |
| `ScoringTests.cs` | Regression coverage for Scoring. |
| `SquadRulesTests.cs` | Regression coverage for Squad Rules. |
| `TestSquad.cs` | Reusable squad and player fixtures for deterministic fantasy rule tests. Defines `TestSquad`. |
| `TransferChipLeagueTests.cs` | Regression coverage for Transfer Chip League. |

### `tests/Touchline.Api.Tests`

| File | Responsibility and connections |
|---|---|
| `SportmonksDecodingTests.cs` | Regression coverage for Sportmonks Decoding. |
| `Touchline.Api.Tests.csproj` | MSBuild framework and NuGet dependency settings for Touchline.Api.Tests. |
