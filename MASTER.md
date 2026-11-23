# Touchline • team knowledge base

Touchline puts South African football in fans’ hands. A supporter can follow a club through its fixtures, results, league position and match statistics. Fantasy gives the same matches a personal stake. A fan picks a squad of real players, chooses a starting eleven, earns points from their performances and competes with other managers each gameweek. The intended experience is one account and one team across devices, with a React Native app available through the Apple App Store and Google Play.

**Baseline • 23 November 2026.** Phatu supplied the product direction, team responsibilities and hosting status. The descriptions of behaviour below come from the local source revisions listed here. The store release, API host and service consolidation are plans. Production dashboards, store consoles and the live database were outside this review.

## Find your starting point

[UI concepts in Figma and offline screen overview](ui-concepts/figma-overview.md)

| Guide | Question it answers | Source checkout and inspected revision |
|---|---|---|
| [Main, UI and mobile release](touchline_main.md) | What does each screen do, where does its data come from, and how do we put it on a phone? | `touchline_main_fresh` • `main@790fc10` |
| [Platform API](touchline_api.md) | Who can read or change a squad, how are points calculated, and what needs hosting? | `touchline_api` • `phathu-dev@f887905` |
| [Football and news sync](touchline_sync.md) | What runs on a schedule and how does a match become usable data? | `touchline` • `main@4c550f7` |
| [Database](DB.md) | Which table owns a fact, which ID joins it, and how does the team safely evolve the schema? | Flyway migrations V001–V038 in `touchline`, compared with application code in both other repositories |
| [Accounts and sign-in](AUTH.md) | Which sign-in options has the team agreed on, and what still needs verification? | Product decision confirmed by Phatu on 23 November 2026. |

Each repository guide ends with a file catalogue. Paths there start at its checkout, so they work as search terms on any teammate’s computer. The database guide explains the SQL history in the sync repository. The five project plans and the UI concept artwork informed the intended direction. For example, the worker plan predates the Python completeness writer and the auth migration now in Flyway. Use the dated baseline when resolving differences. The team vault’s decision register was a template, with no dated approvals.

## The five people building Touchline

| Person | Current responsibility |
|---|---|
| **Max** | UI, including React Native screens, interaction states and how football and fantasy data appear on a phone. |
| **Phatu** | Shared backend development across API, sync, database and integration. |
| **Kanya** | Shared backend development across API, sync, database and integration. |
| **Leruo** | Shared backend development across API, sync, database and integration. |
| **Funi** | Shared backend development across API, sync, database and integration. |

Any of the four backend contributors can take a change across a worker, migration and API route. For a screen change, Max and the contributor doing that work should agree on one concrete response, an empty state and an error case. Name the person driving each decision when the team makes it.

## What communicates today

| Caller | Destination and transport | Information that crosses the boundary |
|---|---|---|
| Netlify sync functions | Sportmonks via HTTPS, then Supabase PostgreSQL via SQL | Scheduled reads of competitions, seasons, teams and fixtures with embedded match details. Upserts carry provider identifiers forward. |
| Netlify RSS job | Publisher feeds via HTTPS, then PostgreSQL | A recent South African football news list stored under one cache key. |
| Netlify-hosted web client | Netlify functions via `/api/v1`. Football reads use Supabase PostgREST. News reads PostgreSQL directly | Football lists, match detail, rankings and cached headlines. The web repository also ships a different fantasy write contract. |
| Expo app | Supabase Auth via HTTPS | Sign-up, sign-in, session restoration and access-token refresh. |
| Expo app | Dedicated ASP.NET API via a configured base URL | Public football reads, authenticated profile, team creation and squad snapshot. Phatu reports the API is still unhosted. A developer machine can serve local testing. |
| ASP.NET API | Supabase signing keys via HTTPS and PostgreSQL via Npgsql | Verifies the fan’s token, reads provider facts and writes fantasy records as the API role. The sync handoff is through committed database rows. |
| Expo News tab | Node/Express news service via a second URL | Categories and articles from its Prisma database. A production deployment of this service is unverified. |
| Alternative Python worker | Sportmonks and PostgreSQL, plus its own internal HTTP operations | Ingests football, journals jobs and derives completeness flags. A Render blueprint exists. The team reports Netlify is hosting sync. |

## Follow one supporter through a gameweek

1. **Before the match.** Sync loads a PSL season, two clubs and a fixture. Kickoff is stored as an instant in UTC. On mobile, Live Games uses the fixtures and teams endpoints. The PSL table groups real clubs and counts completed scores. A fantasy league groups managers and counts points from their picks.
2. **Join the game.** Supabase issues a token. The API validates it, resolves the UUID to an internal profile and creates one fantasy team for that user and season. The UI presents players with internal IDs, inferred club, position and current price. Selecting players locally is a draft until the API accepts the squad.
3. **Pick and lock.** The configured budget is R300m. Fifteen picks become eleven starters and four substitutes. The API validates the full squad and applies its deadline, normally 90 minutes before the first kickoff. A saved gameweek records the formation, captaincy, bench order and budget. The existing phone UI covers only part of this journey.
4. **Watch the result.** A later sync updates the score, lineup and events. A goal is attached to a provider player ID. The API translates it to the internal player on the fantasy pick. Counts and timestamps depend on ingestion cadence, so a match marked live can still have an old score.
5. **Publish points.** The intended result is a breakdown the supporter can explain. An admin finalisation writes player results and a team total, from which league rankings are calculated. The checkout still guards minute-dependent scoring until starter and substitution mappings are confirmed. A second device should see the same persisted result.

## From today’s hosting to a store release

**Recommended first milestone:** on a real Android phone and iPhone, sign in against staging, create a team, save 15 legal picks, reopen it on the other device, then score a known completed fixture and reconcile the rank. This tests identity, data, game rules and distribution together.

| Order | Work and evidence to keep |
|---|---|
| 1 • Establish the baseline | Record the active Netlify production branch and jobs, last successful fixture and article refresh, target Supabase project, Flyway history and store listing/signing ownership. Compare the release branch with the source revisions above. |
| 2 • Bring up staging | Keep the web and sync on their reported Netlify deployments. Deploy the dedicated .NET 10 API as an HTTPS service beside them. Configure a scoped database role, the matching Auth project, health checks and explicit web origins. Give Expo a reachable API URL and a working news endpoint. |
| 3 • Prove one gameweek | Rehearse migrations on clean and populated staging, confirm fixture/season selection, validate player-minute codes and choose the completeness rule. Hand-calculate a score, repeat finalisation and inspect a league tie. |
| 4 • Connect both clients | Complete the mobile pick/save and league flows. Replace web-local identity, reconcile any existing teams and move fantasy writes to one endpoint. Migrate web reads feature by feature after verifying shapes and old-build compatibility. |
| 5 • Distribute | Link the team Expo project, confirm Apple and Google developer access, test release-signed builds, complete privacy/deletion and review information, then submit through the stores. Netlify keeps serving remote services while installed apps update on their own schedule. |

## Decisions and open questions

| Status on 23 Nov 2026 | Subject | Basis or next evidence |
|---|---|---|
| Team direction | South African football information and weekly fantasy, shipped to iOS and Android through React Native and Expo | Phatu’s project brief. Max owns UI. Backend work is shared. |
| Team-reported current state | Web and sync hosted on Netlify, dedicated API unhosted, Expo account exists | Record deployment IDs, Expo project owner and store account access when verified. |
| Open | API host and environment URLs | Pick a managed .NET deployment after checking region, DB connectivity, logs, cost and staging rollback. [API delivery](touchline_api.md) |
| Open | Football writer and freshness promise | Retain Netlify with parity work, or hand football to the Python worker after staging comparison. Decide matchday refresh target and responsibility for rescoring corrections. [Sync](touchline_sync.md) |
| Open | Budget and scoring copy | API-created teams start at R300m. SQL defaults to R100m. Confirm the intended default and the captain/chip multipliers before publishing game rules. [Database](DB.md) |
| Open | Shared read contract for news, match detail and standings | The plans favour one public API. Identify older clients and demonstrate parity before switching each route. [Main](touchline_main.md) |
| Agreed · 23 November 2026 | Sign-in options | Continue with Facebook and Sign in with Apple. See the [accounts guide](AUTH.md) for the decision and remaining implementation checks. |
| Open | Store listing continuity and v1 scope | Verify whether the existing Android bundle has been distributed. Confirm the finished screens, account deletion path and news host for the first release. [Main](touchline_main.md) |

## Keeping these six files current

Keep the folder together in the team’s shared, version-controlled documentation location. Use this master as the entry point. A contributor who changes a route, table, rule, screen or deployment updates its guide with the same change. A cross-repository change links the corresponding pull requests, increases the documented migration baseline if needed and updates each affected guide. Add a short dated decision here whenever behaviour or direction changes. Keep prior decisions visible by marking them superseded.

| When a change touches… | Update… | Include evidence a teammate can repeat |
|---|---|---|
| Football IDs, status codes, fixture timing or imports | Sync + DB + API, and Main if the display changes | A sample provider ID, internal ID, before/after fields and expected rendered state. |
| Identity, fantasy rules or an endpoint | API + Main, AUTH for sign-in changes, and DB for persistence changes | One request, expected response or validation error and test reference. |
| Native navigation, build settings, deployment or distribution | Main + this guide for decisions affecting the release | Environment name, build number, supported version and smoke-test result. |
| A new or renamed tracked file | Its repository catalogue | Relative path, job and callers. Update the baseline commit when verified. |

**Decision record to copy**

```markdown
### YYYY-MM-DD • Decision title
Status — Proposed / Agreed / Superseded by …
Decision and rationale — What changes for fans or contributors, and why.
People — Driver, reviewers and date agreed.
Affected systems — Repositories, routes, schemas, screens and minimum app version.
Delivery — Migration or rollout order, compatibility window and recovery plan.
Evidence — PRs, migration version, test or staging observation.
Guides updated — Links and verified commit or deployment.
```

An agreed design can precede deployment. Record the environment and verification date separately so the next teammate can tell which behaviour a fan actually receives.
