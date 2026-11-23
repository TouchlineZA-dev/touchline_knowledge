# Touchline platform API plan

**Program** — `/Users/phathusigama/Documents/touchline_api`, local `phathu-dev` at `9fbb5bc`.  
**Outcome** — Web, Capacitor and Expo share one public `/api/v1` contract. Supabase Auth establishes identity. ASP.NET validates the bearer token, resolves the profile and authorizes each fantasy action.

## Keep the existing API as the foundation

`Program.cs`, `TouchlineDbContext.cs`, `Controllers/FantasyControllers.cs` and the services under `src/Touchline.Api` already cover football reads, user provisioning, team creation, squads, transfers, chips, leagues, points and admin operations. The embedded ASP.NET project in the client repo has a different persistence model and port. Use it as a reference when checking legacy behavior. Deploy the dedicated repo.

The inspected branch is two commits ahead of the locally recorded `origin/main` and contains auth work and integration tests. Review and promote that work through CI. The working tree also has a local change to `appsettings.Development.json`. Keep deployment settings in the environment and review that edit separately before committing.

## Deliver the first end to end flow

1. **Build against the released Flyway schema.** Coordinate the [sync migration work](01-sync-worker.md). `tests/Touchline.Api.Tests/Auth/PostgresFixture.cs` reads V033 from a deleted file. Change the fixture to consume the canonical migration release. Run auth and fantasy persistence tests on disposable Postgres with the real grants. Exercise a populated upgrade and clean install.
2. **Establish one identity.** Use the same environment-specific Supabase project as both clients. Verify issuer, audience, signature and token subject. Preserve the API’s 409 recovery for legacy and conflicting emails. Add a profile update contract for display name and favourite club, and a team rename action if retaining those web features. Pick stable internal club IDs and define how existing club selections are mapped.
3. **Prove a team is shared.** Create the current PSL season’s gameweek through the admin route using synced fixtures and an editable deadline. Sign in, call `GET /me`, create a team with `POST /fantasy/team`, submit 15 picks with `PUT /fantasy/team/squad` and load `GET /fantasy/team/snapshot` on the other client. Use the configured R300m starting budget and check the returned balance.

## Complete the public contract

| Consumer need | Change in this API |
| --- | --- |
| Web fixtures and players | Agree on `kickoffUtc`, status, crests, scorers and nullable fields. The web expects `kickoff` and player ratings absent from C#. Serve the facts the screens need and retire synthetic ratings or label a separately defined rating. |
| PSL table, player rankings and match detail | Port the behavior exercised by the web functions for `/standings`, `/player-rankings` and `/matches?fixtureId=`. Preserve competition and season filters, ranking tie rules, events, stats and lineups. Use the same table on mobile. |
| Headlines and category navigation | Expose one article and category contract through C#. Keep compatibility for `/news/headlines?max=` during rollout. Adapt the private news service’s data and errors. See the [news plan](05-news-service.md). |
| Fantasy edits | Publish request and response examples from the implemented controllers. Preserve structured 400 validation errors, 401 sign-in, 403 permissions, 404 missing team, 409 conflict and 503 unavailable data. Expose `ETag` to permitted browser origins. Make version checks atomic on concurrent lineup and transfer writes, and test two clients editing the same team. |

Export OpenAPI from a development or CI build and version it for both clients. Exercise actual response envelopes in contract tests. Publish a local launch target and staging/production deployment configuration for the dedicated .NET 10 service, using the scoped database role, the correct Supabase settings and explicit allowed origins. Monitor database readiness and data freshness separately.

## Make scoring and prices trustworthy

`Business/SportmonksCodes.cs` leaves starter, bench and substitution direction unset. The guarded scoring and valuation paths can return 503. Use the worker’s representative PSL fixtures to confirm these mappings. Hand-check minutes, a substitute, captain points and a corrected goal. Assert expected results in tests before enabling scoring and price runs.

Agree how the finalizer uses `sync.gameweek_sync_status`. The current API can finalize when all fixture states are final even if event ingestion is incomplete. Set a completeness rule for late facts and postponements. Give operations an explicit owner or scheduled invocation for creating gameweeks, finalizing or rescoring them and applying reviewed transfer-window prices. Test repeated scoring, duplicate transfer requests, league capacity and two-device edits against Postgres.

**Release gate** — Both clients read the same team ID, picks and balance. A stale edit gets a recoverable conflict. A non-admin cannot score. A final score and league rank match a hand-calculated fixture and remain stable after a rerun. Keep previous reads available during rollout and keep a single writer for each team.

**Local check** — A clean copy of commit `9fbb5bc` built with zero errors or warnings. All 174 tests outside the Auth test namespace passed. This run excluded auth integration tests and used no live Supabase project. The missing V033 test input and the cross-program staging checks remain open.
