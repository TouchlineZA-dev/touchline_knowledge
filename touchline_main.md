# Touchline Main • UI, clients and store delivery

[Team home](MASTER.md) · [Platform API](touchline_api.md) · [Sync worker](touchline_sync.md) · [Database](DB.md)

**Source • `touchline_main_fresh`, `main@790fc10`, 23 November 2026.** Max leads the UI. Phatu, Kanya, Leruo and Funi share backend work, including the contracts on which these screens depend. This guide covers both the existing Netlify web experience and the Expo/React Native application intended for iOS and Android.

## Which application am I looking at?

| Directory | Entry point and runtime | Service it talks to |
|---|---|---|
| `mobile-app/` | Expo Router → `app/_layout.tsx` → five tabs and account modal. React Native and Metro. | Supabase Auth, dedicated .NET football/fantasy API, separate Node news API. |
| `frontend/` | `index.html` → `src/main.tsx` → `App.tsx` → Index → FootballDashboard. React DOM, Vite and Tailwind. | Relative `/api/v1` on Netlify by default. The functions talk to Supabase. |
| `frontend/android/` | Capacitor’s Android shell packages the web build. A Gradle project, upload keystore file and existing AAB are tracked. | Web client code has a native-host branch with a hard-coded Netlify API origin. Confirm installed versions before retiring routes. |
| `touchline-news-api/` | Express server, Prisma and PostgreSQL. An optional scheduled job ingests articles. | GNews and its configured database. Expo calls its public read endpoints. |
| `Touchline API/` | Embedded ASP.NET project with its own launch profile (5080), Supabase REST client and SQL proposal. | Different fantasy API using GUID-based identity and RPCs. The intended central service lives in the dedicated checkout. |

## Inside the native UI

The root layout loads fonts, holds the splash until ready, establishes the Supabase auth context and opens the tab stack. The tab layout names News, My Squad, Leagues, PSL Table and Live Games. The `two` starter route is hidden from navigation. The modal implements account entry. Screen loaders use `useApi`, which tracks loading, error and the latest call so an old response cannot replace a newer one.

| Screen | Implemented path | Work to reach the product experience |
|---|---|---|
| News | `app/(tabs)/index.tsx` gets categories and up to 20 articles from Node. It switches category, refreshes and opens publisher links. | Compare the slug sent by the screen with the service’s case-insensitive equality check on the article’s category string. Verify a multiword category and a genuine empty feed. |
| My Squad | `squad.tsx` reads the profile and snapshot, can look up overall standings, handles no team, creates a team and browses/searches players. The pitch and formation chooser include local screen state. | Wire an editable 15-player draft, starters, ordered bench and captaincy to the first squad save. Then implement lineup edits, transfers and chips with deadlines, budget, server errors and version handling. |
| Leagues | `leagues.tsx` displays a coming-soon banner. The client already defines list, create, join and standings calls. | Build create/join/leave, owner-only invite code, capacity and membership errors, and show weekly vs overall rank. Add a leave client method. |
| PSL Table | `table.tsx` gets fixtures, teams and season. `football.ts` counts finished fixtures with both scores present. A win awards three, a draw one. Sort by points, goal difference, goals scored and club name. | Compare this derived table with the web standings response, including postponed fixtures, official adjustments if relevant, and clubs on zero games. Adopt a shared contract after parity. |
| Live Games | `live.tsx` combines fixtures and team crests, groups matches by SAST day and supports refresh. It shows live games and the window from three days back to 14 days ahead, falling back to the next ten upcoming fixtures. | Expose freshness and connect full match detail (events, stats and lineups) through a supported API. Preserve the distinction between “0–0”, “no score yet” and “feed delayed”. |
| Account modal | Native email/password sign-up and sign-in, session restore, sign-out. The modal’s social sign-in handler currently directs native users to email. | Verify confirmation and recovery links on physical devices. If shipping social sign-in, configure native callbacks and provider behaviour. Implement account deletion and test what happens to the manager’s data. |

### Visual references and reusable UI

The implemented mobile palette in `src/theme/design.ts` uses green `#2E9E4F` for core branding, white cards on `#E6E6E6`, pitch greens and distinct bench colours. Bebas Neue is used for the wordmark, Lexend for headings and Open Sans for body text. `TouchlineHeader`, `Common` and `ScreenState` provide shared header, pill, crest and feedback components. Prefer those when shipping a new screen so a missing crest, retry action or title behaves consistently across tabs.

The five `mobile-app/figma-*.png` files are screen references. The project also has the Home/News, My Squad and Match Centre studies plus seven glass-style concepts from Welcome through Live Games. These are possible treatments, and the source tokens above identify what currently renders. The redesign blueprint asks designers to preserve club/player names, abbreviations and readable text at varying lengths. For the pick-team screen, test long surnames, a large system font, a four-player bench and the error message for a fourth player from one club. Put captain/vice labels and fixture status in text as well as colour. Verify touch targets, safe areas, screen-reader order, and back navigation on both operating systems.

### Network boundary and state

`src/lib/config/env.ts` reads `EXPO_PUBLIC_API_BASE_URL` (include `/api/v1`), `EXPO_PUBLIC_NEWS_API_BASE_URL` (include `/api`), `EXPO_PUBLIC_SUPABASE_URL` and `EXPO_PUBLIC_SUPABASE_ANON_KEY`. The project URL and placeholder key have fallbacks in source. Make release builds validate explicit values. Every `EXPO_PUBLIC_*` value is client-visible. Put only public origins and the publishable/anon Auth client key there.

The mobile Supabase client persists the session through SecureStore on native and local storage on its web target. It auto-refreshes tokens. The API client asks for the current access token on protected requests, sends `Authorization: Bearer …`, aborts after 20 seconds, and handles JSON, 204 and service errors. It currently returns the decoded body and drops headers. Before enabling multi-device lineup edits, preserve the ETag from squad/snapshot GETs, send `If-Match` and handle 304/409 explicitly. Scope any saved private snapshot to account and season and clear it on sign-out.

A practical handoff for Max: render the next gameweek’s deadline, display the **server-returned** remaining budget, disable submission while saving, preserve unsaved edits if validation fails, and refresh the snapshot after an accepted write. The API uses internal player IDs. A crest asset, display name or provider ID cannot identify a purchased pick.

## Web and Netlify

The root mounts routing, club theme, auth and query providers. `FootballDashboard` composes football views such as standings and match detail. `src/services/touchlineApi.ts` selects a relative API base in a browser and a fixed Netlify origin for its recognised native web-shell case. `frontend/netlify.toml` rewrites each supported API route to a function before the single-page fallback. An unknown API path can therefore resolve to HTML. Check content type when debugging a JSON parse failure.

| Web function | Contract served today |
|---|---|
| `_supabase.js` | Creates a server-side Supabase client, normalises the project origin and pages large reads past the PostgREST row limit. Sensitive keys stay in server environment settings. |
| `fixtures.js`, `matches.js` | List fixtures with web-specific fields, then get match detail including events, statistics and lineups. The web model expects `kickoff`, crests and scorers. |
| `players.js`, `player-rankings.js`, `standings.js` | Build the player catalogue, statistical leaders and real-club table. The web player response includes generated ratings. The dedicated API returns a different player model. |
| `news.js` | Uses a server-side `pg` pool and `DATABASE_URL` to read the `sa_football_news` cache directly. Converts snake_case image/date fields to the web model. Missing configuration can return HTTP 200 with `unavailable: true` and an empty list. |
| `health.js`, `fantasy.js` | Web-specific health and legacy fantasy operations. The fantasy function expects different tables/RPCs, including `fantasy_team_picks` and `save_fantasy_team`. |

**Web identity is a release dependency.** `authService.ts` stores local accounts, including the submitted password unchanged in a field called `passwordHash`. A browser-local ID differs from a Supabase subject. Move web registration/sign-in to Supabase and resolve the profile through the platform API. Remove the legacy stored credentials and require fresh authentication. Migrate eligible non-secret preferences or picks only after verifying ownership. `fantasySquadService.ts` also contains browser-local fantasy state and rules. Do not infer that a web draft is persisted on a phone.

During cutover, retain the current web reads until fixture/standings/detail/news shapes agree. If the legacy function holds real teams, reconcile team counts, ownership and purchases before routing all writes to the dedicated API. Keep older installed clients in the compatibility window. The Android project and AAB in this checkout are evidence of packaging work. They cannot establish whether a Play listing is live.

## Node news service

`src/server.ts` opens the Express listener and, when enabled with a GNews key, schedules periodic ingestion. The provider retrieves stories. The sync service normalises and deduplicates, classifies by category, matches team aliases and writes articles plus an ingestion log through Prisma. The API mounts news, category, source and team controllers, rate limiting, errors and Swagger documentation. The health route reports process status. Use a real article query to verify database connectivity.

Read endpoints use `/api`: `/news`, `/news/:id`, `/news/search?q=…`, `/news/category/:category`, `/categories`, `/sources`, `/teams` and `/teams/:team/news`. Listing defaults to page 1 and 20 articles, with a 100 limit cap. Compare timestamps, source URL, category string and team tags against mobile types in `src/lib/api/types.ts`. The proposed central API would adapt these reads and keep the web headline route compatible. Staging needs a working second URL until that exists.

## Local development

| Starting directory | Commands and reachable services |
|---|---|
| `mobile-app/` | Install the locked npm dependencies with `npm ci`. Set the public environment above. Run `npm start`, `npm run android` or `npm run ios`. Check types with `npx tsc --noEmit`. The local check on 23 November reproduced TS2322 in `components/ExternalLink.tsx:11`: a plain string is passed to a typed Expo Router link. Resolve the link type and repeat the check on the release revision. |
| `frontend/` | `npm ci`, `npm run dev` starts Vite on 8080. The existing development proxy forwards `/api` to 8000 and function URLs to 8888. Set a deliberate base for C# on 5147. `npm run build` produces `dist`. The Netlify file sets Node 22. |
| `touchline-news-api/` | Use the package scripts from this directory. Configure an isolated PostgreSQL URL and provider settings, install with `npm ci`, generate Prisma client if needed (`npx prisma generate`), apply its migrations (`npm run migrate`), seed, sync and `npm run dev`. `npm test` runs Vitest. Local default service port is 3000. |

On an iOS simulator the development host can be localhost. The Android emulator defaults to `10.0.2.2`. A physical phone needs a reachable development host or staging HTTPS address for each service. Test on cellular as well as Wi-Fi before calling a build store-ready.

## Expo, Apple App Store and Google Play release plan

**Release target.** Build from `mobile-app/`. Its config declares Touchline SA, slug `touchline-sa`, scheme `touchlinesa`, version 1.0.0 and iOS/Android identifiers `za.co.touchlinesa.app`. The same Android application ID exists in Capacitor. Verify the Play listing, highest version code, signing/upload certificate and who controls it before creating a new app record or signing an update. The tracked upload-keystore artifact warrants an access and key-custody review. Keep signing material out of the knowledge base. The team has an Expo account. No tracked `eas.json` or explicit EAS project ID appears in this app snapshot.

1. **Establish team ownership.** Confirm the Expo organisation/project, Apple Developer team and App Store Connect access, and Google Play Console developer account and app ownership. Link the mobile directory to the intended EAS project. Record identifiers, roles and recovery contacts privately. Configure development, preview/staging and production build profiles with explicit service URLs.
2. **Deploy the remote services.** Keep Netlify serving web and its scheduled workers. Give the dedicated .NET API a stable public HTTPS hostname. Host the Node news service or complete the central news adapter. Configure the matching Auth project and callback allowlist. Verify `/api/v1/health`, a football read, authenticated `/me` and a fresh news read from a device.
3. **Build a release candidate.** From the linked directory, use `eas build:configure`. After creating the profiles, run `eas build --profile preview --platform all` for device testing and `eas build --profile production --platform all` for store binaries. Choose version/build numbers above any previously distributed build. Use EAS-managed or team-controlled signing credentials consistent with the existing listing. [Expo store builds](https://docs.expo.dev/deploy/build-project/)
4. **Exercise the supporter journey.** Fresh install, sign-up/confirmation and returning sign-in, read fixtures in SAST, complete a lawful team under budget, change captain, handle stale edits and a locked deadline, join a league, recover from a lost network and sign out. Compare the same account on iOS and Android. Check a no-team account and a reviewer account with a populated squad even between matches.
5. **Prepare store information.** Supply screenshots of shipping screens, support and privacy URLs, descriptions, age/content entries and accurate disclosures for account, analytics and SDK data. Provide a working reviewer account or demo mode. Account creation requires an accessible deletion flow on iOS. Google also calls for an in-app path and a functional external deletion-request link disclosed in Play Console. Define the backend process for Auth identity, app profile and fantasy records before that UI ships. [Apple review guidelines](https://developer.apple.com/app-store/review/guidelines/) · [Google Play account deletion](https://support.google.com/googleplay/android-developer/answer/13327111)
6. **Submit and promote.** Upload the signed iOS build to App Store Connect/TestFlight, test it, select it for App Review and choose the release timing. Upload the Android App Bundle to Play internal testing, meet the account’s applicable testing and production-access requirements, and promote through the intended track. EAS Submit can upload builds, and manual store uploads are available. A successful EAS upload is an intermediate milestone. Record the store-approved version and rollout status for each platform. [Expo submission guide](https://docs.expo.dev/deploy/submit-to-app-stores/) · [Google Play testing requirements by account](https://support.google.com/googleplay/android-developer/answer/14151465)
7. **Operate installed versions.** Monitor API errors, sign-in, match freshness and crashes by app version. Keep older read shapes available during rollout. Roll the backend back to a compatible revision if needed, pause store rollout for a bad binary, and ship a higher-numbered corrected build. Record the minimum supported app version before removing an endpoint.

**Definition of ready for the team:** Max can demonstrate the actual screens from an installed build, a backend contributor can trace one saved pick to its database row and explain its points, and both store test tracks can reach the same intended environment.

## File catalogue

This revision contains **508 tracked files**. Combine each directory heading with the filename beneath it. The entries name responsibilities and selected declarations or direct source dependencies. Installed libraries, Git metadata, caches and intermediate builds are local machine state. The private `.env` supplies runtime settings. Share setting names through the example files.

### Repository root

| File | Responsibility and connections |
|---|---|
| `.env.example` | Lists environment setting names and formats. Supply actual values privately per environment. |
| `.gitignore` | Git exclusions for local settings and generated files. |
| `BACKEND_INTEGRATION.md` | Repository reference for backend integration. Compare its assumptions with the inspected code. |
| `IMPLEMENTATION_PLAN.md` | Repository reference for implementation plan. Compare its assumptions with the inspected code. |
| `README.md` | Repository reference for . setup and use. Compare its assumptions with the inspected code. |
| `TOUCHLINE_NEWS_API_HANDOFF.md` | Repository reference for touchline news api handoff. Compare its assumptions with the inspected code. |
| `Touchline_UI_Redesign_Blueprint.docx` | Design handoff on localising layouts, preserving football terms and testing text expansion. |
| `touchline-news-api-handoff.zip` | Packaged news-service handoff. The unpacked service is available for source review. |

### `Touchline API/Database`

| File | Responsibility and connections |
|---|---|
| `FantasySchemaProposal.sql` | Alternative RPC-based schema for the embedded API and legacy fantasy path. |

### `Touchline API`

| File | Responsibility and connections |
|---|---|
| `FantasyService-from-api-repo.cs` | Reference copy of services from the API, outside the embedded web project directory. Defines `FantasyService`, `FantasyConflictException`, `FantasyAuthenticationException`. |
| `README.md` | Repository reference for Touchline API setup and use. Compare its assumptions with the inspected code. |

### `Touchline API/Touchline.Api/Controllers`

| File | Responsibility and connections |
|---|---|
| `FantasyController.cs` | HTTP boundary for fantasy operations. Dispatches to the service layer. Defines `FantasyController`. |
| `PlayersController.cs` | HTTP boundary for players operations. Dispatches to the service layer. Defines `PlayersController`. |

### `Touchline API/Touchline.Api/Infrastructure`

| File | Responsibility and connections |
|---|---|
| `SupabaseRestClient.cs` | Wraps PostgREST reads and database RPC calls for the embedded C# service. Defines `SupabaseRestClient`, `SupabaseRequestException`. |
| `SupabaseSettings.cs` | Defines Supabase endpoint and key settings for the embedded service. Defines `SupabaseSettings`. |
| `SupabaseTokenValidator.cs` | Validates the caller’s token for the embedded API identity model. Defines `SupabaseTokenValidator`. |

### `Touchline API/Touchline.Api/Middleware`

| File | Responsibility and connections |
|---|---|
| `ApiExceptionMiddleware.cs` | Formats embedded API errors into HTTP responses. Defines `ApiExceptionMiddleware`. |
| `SupabaseAuthMiddleware.cs` | Resolves the embedded API caller and stores the user identity for its controllers. Defines `SupabaseAuthMiddleware`. |

### `Touchline API/Touchline.Api/Models`

| File | Responsibility and connections |
|---|---|
| `FantasyRequests.cs` | Legacy request types for fantasy actions and persisted selections. Defines `SaveFantasyTeamRequest`, `FantasyPickRequest`, `ActivateChipRequest`. |
| `ScoringModels.cs` | Types consumed by the embedded scoring implementation. Defines `PlayerMatchEvent`, `PlayerScoreBreakdown`, `FantasyPickForScoring`. |

### `Touchline API/Touchline.Api`

| File | Responsibility and connections |
|---|---|
| `NuGet.Config` | Defines package sources for restoring the embedded .NET project. |
| `Program.cs` | Configures the embedded ASP.NET service, separate from the dedicated EF Core API. Defines `Program`. |
| `Touchline.Api.csproj` | MSBuild framework and NuGet dependency settings for Touchline.Api. |
| `Touchline.Api.http` | Example HTTP requests for the embedded project’s development server. |
| `appsettings.Development.json` | ASP.NET development overrides loaded at startup. |
| `appsettings.json` | ASP.NET base configuration loaded at startup. |

### `Touchline API/Touchline.Api/Properties`

| File | Responsibility and connections |
|---|---|
| `launchSettings.json` | Defines the embedded API launch profiles including port 5080. |

### `Touchline API/Touchline.Api/Services`

| File | Responsibility and connections |
|---|---|
| `FantasyRules.cs` | Embedded API version of fantasy limits and settings. Defines `FantasyRules`, `FantasyRuleException`. |
| `FantasyScoringService.cs` | Calculates player and squad outcomes within the embedded API implementation. Defines `FantasyScoringService`. |
| `FantasyService.cs` | Implements embedded fantasy reads and writes using Supabase REST and database functions. Defines `FantasyService`, `FantasyAccessException`. |

### `frontend`

| File | Responsibility and connections |
|---|---|
| `.env.example` | Lists environment setting names and formats. Supply actual values privately per environment. |
| `.firebaserc` | Firebase CLI project selection for the older web deployment setup. |
| `ADMIN_SYSTEM.md` | Repository reference for admin system. Compare its assumptions with the inspected code. |
| `ALL_PLAYERS_LIST.md` | Repository reference for all players list. Compare its assumptions with the inspected code. |
| `APP_LOGIC.md` | Repository reference for app logic. Compare its assumptions with the inspected code. |
| `DATA_FLOW_UML.md` | Repository reference for data flow uml. Compare its assumptions with the inspected code. |
| `FANTASY_API.md` | Repository reference for fantasy api. Compare its assumptions with the inspected code. |
| `IMPLEMENTATION_STATUS.md` | Repository reference for implementation status. Compare its assumptions with the inspected code. |
| `README.md` | Repository reference for frontend setup and use. Compare its assumptions with the inspected code. |
| `app-release.aab` | Existing packaged Android release artifact from the web/Capacitor line. |
| `bun.lockb` | Alternative package-resolution lock from Bun. |
| `capacitor.config.ts` | Maps application identity and web output directory into Capacitor’s native shell. |
| `components.json` | UI primitive generator configuration for the web component library. |
| `eslint.config.js` | Web lint rules and TypeScript/React development checks. |
| `firebase.json` | Legacy Firebase hosting configuration for the web build. |
| `game_week.csv` | Legacy game week dataset for historical import/reference. |
| `index.html` | Vite HTML entry and mount point for the web React root. |
| `manualSyncNews.ts` | Legacy manual news/Firebase sync script referencing older integration modules. |
| `netlify.toml` | Sets the web build, functions directory and ordered API/SPA rewrites. |
| `package-lock.json` | Pins npm package resolutions for reproducible installs. |
| `package.json` | Declares libraries and runnable scripts for frontend. |
| `player_stats.csv` | Legacy player stats dataset for historical import/reference. |
| `players.csv` | Legacy players dataset for historical import/reference. |
| `pnpm-lock.yaml` | Alternative pnpm dependency-resolution lock. |
| `pnpm-workspace.yaml` | Defines pnpm workspace settings for web tooling. |
| `postcss.config.js` | Runs the CSS transformation plugins for web styles. |
| `psl_standings.csv` | Legacy psl standings dataset for historical import/reference. |
| `tailwind.config.ts` | Sets web theme tokens, content scanning and Tailwind extensions. |
| `test-prod-news.js` | Manual smoke check of the configured headlines endpoint. |
| `test-system-fixes.js` | Manual news filtering and application behaviour checks. |
| `test-team-building.js` | Exercises a mock team-building flow with generated player data. |
| `testNews.ts` | Legacy news-fetch harness referring to an earlier web news module. |
| `test_jina.js` | Historical external search experiment containing an embedded credential-like literal. Review its status privately before reuse. |
| `tsconfig.app.json` | TypeScript compile scope, target and resolution settings for this build. |
| `tsconfig.json` | TypeScript compile scope, target and resolution settings for this build. |
| `tsconfig.node.json` | TypeScript compile scope, target and resolution settings for this build. |
| `user_teams.csv` | Legacy user teams dataset for historical import/reference. |
| `users.csv` | Legacy users dataset for historical import/reference. |
| `vite.config.ts` | Runs the dev server on 8080 and routes local API traffic through configured proxies. |

### `frontend/android`

| File | Responsibility and connections |
|---|---|
| `.gitignore` | Git exclusions for local settings and generated files. |
| `build.gradle` | Root Android buildscript, repositories and plugin dependency versions. |
| `capacitor.settings.gradle` | Includes native Capacitor dependencies in Gradle project resolution. |
| `gradle.properties` | Project-wide properties for Android Gradle builds. |
| `gradlew` | Unix launcher for the project’s pinned Gradle wrapper. |
| `gradlew.bat` | Windows launcher for the project’s pinned Gradle wrapper. |
| `settings.gradle` | Names and includes projects for the Android build. |
| `variables.gradle` | Shared Android SDK and library version settings. |

### `frontend/android/app`

| File | Responsibility and connections |
|---|---|
| `.gitignore` | Git exclusions for local settings and generated files. |
| `build.gradle` | Android app module, package ID, release settings, version and dependencies. |
| `capacitor.build.gradle` | Generated Capacitor module wiring used by the app build. |
| `proguard-rules.pro` | Android shrinking and obfuscation rules. |
| `touchline-upload.jks` | Tracked Android upload-keystore artifact. Confirm certificate ownership and signing continuity in Play Console. |

### `frontend/android/app/src/androidTest/java/com/getcapacitor/myapp`

| File | Responsibility and connections |
|---|---|
| `ExampleInstrumentedTest.java` | Template Android test that runs on a device/emulator. |

### `frontend/android/app/src/main`

| File | Responsibility and connections |
|---|---|
| `AndroidManifest.xml` | Declares the native app, activity, permissions and platform metadata. |

### `frontend/android/app/src/main/java/za/co/touchlinesa/app`

| File | Responsibility and connections |
|---|---|
| `MainActivity.java` | Starts the Capacitor activity that displays the bundled web UI. |

### `frontend/android/app/src/main/res`

| File | Responsibility and connections |
|---|---|
| `drawable-land-hdpi/splash.png` | Android launch artwork for the drawable-land-hdpi device configuration. |
| `drawable-land-ldpi/splash.png` | Android launch artwork for the drawable-land-ldpi device configuration. |
| `drawable-land-mdpi/splash.png` | Android launch artwork for the drawable-land-mdpi device configuration. |
| `drawable-land-night-hdpi/splash.png` | Android launch artwork for the drawable-land-night-hdpi device configuration. |
| `drawable-land-night-ldpi/splash.png` | Android launch artwork for the drawable-land-night-ldpi device configuration. |
| `drawable-land-night-mdpi/splash.png` | Android launch artwork for the drawable-land-night-mdpi device configuration. |
| `drawable-land-night-xhdpi/splash.png` | Android launch artwork for the drawable-land-night-xhdpi device configuration. |
| `drawable-land-night-xxhdpi/splash.png` | Android launch artwork for the drawable-land-night-xxhdpi device configuration. |
| `drawable-land-night-xxxhdpi/splash.png` | Android launch artwork for the drawable-land-night-xxxhdpi device configuration. |
| `drawable-land-xhdpi/splash.png` | Android launch artwork for the drawable-land-xhdpi device configuration. |
| `drawable-land-xxhdpi/splash.png` | Android launch artwork for the drawable-land-xxhdpi device configuration. |
| `drawable-land-xxxhdpi/splash.png` | Android launch artwork for the drawable-land-xxxhdpi device configuration. |
| `drawable-night/splash.png` | Android launch artwork for the drawable-night device configuration. |
| `drawable-port-hdpi/splash.png` | Android launch artwork for the drawable-port-hdpi device configuration. |
| `drawable-port-ldpi/splash.png` | Android launch artwork for the drawable-port-ldpi device configuration. |
| `drawable-port-mdpi/splash.png` | Android launch artwork for the drawable-port-mdpi device configuration. |
| `drawable-port-night-hdpi/splash.png` | Android launch artwork for the drawable-port-night-hdpi device configuration. |
| `drawable-port-night-ldpi/splash.png` | Android launch artwork for the drawable-port-night-ldpi device configuration. |
| `drawable-port-night-mdpi/splash.png` | Android launch artwork for the drawable-port-night-mdpi device configuration. |
| `drawable-port-night-xhdpi/splash.png` | Android launch artwork for the drawable-port-night-xhdpi device configuration. |
| `drawable-port-night-xxhdpi/splash.png` | Android launch artwork for the drawable-port-night-xxhdpi device configuration. |
| `drawable-port-night-xxxhdpi/splash.png` | Android launch artwork for the drawable-port-night-xxxhdpi device configuration. |
| `drawable-port-xhdpi/splash.png` | Android launch artwork for the drawable-port-xhdpi device configuration. |
| `drawable-port-xxhdpi/splash.png` | Android launch artwork for the drawable-port-xxhdpi device configuration. |
| `drawable-port-xxxhdpi/splash.png` | Android launch artwork for the drawable-port-xxxhdpi device configuration. |
| `drawable-v24/ic_launcher_foreground.xml` | Android ic launcher foreground resource used by the Capacitor app. |
| `drawable/ic_launcher_background.xml` | Android ic launcher background resource used by the Capacitor app. |
| `drawable/splash.png` | Android launch artwork for the drawable device configuration. |
| `layout/activity_main.xml` | Android activity main resource used by the Capacitor app. |
| `mipmap-anydpi-v26/ic_launcher.xml` | Android ic launcher resource used by the Capacitor app. |
| `mipmap-anydpi-v26/ic_launcher_round.xml` | Android ic launcher round resource used by the Capacitor app. |
| `mipmap-hdpi/ic_launcher.png` | Android launcher ic launcher artwork selected for mipmap-hdpi. |
| `mipmap-hdpi/ic_launcher_background.png` | Android launcher ic launcher background artwork selected for mipmap-hdpi. |
| `mipmap-hdpi/ic_launcher_foreground.png` | Android launcher ic launcher foreground artwork selected for mipmap-hdpi. |
| `mipmap-hdpi/ic_launcher_round.png` | Android launcher ic launcher round artwork selected for mipmap-hdpi. |
| `mipmap-ldpi/ic_launcher.png` | Android launcher ic launcher artwork selected for mipmap-ldpi. |
| `mipmap-ldpi/ic_launcher_background.png` | Android launcher ic launcher background artwork selected for mipmap-ldpi. |
| `mipmap-ldpi/ic_launcher_foreground.png` | Android launcher ic launcher foreground artwork selected for mipmap-ldpi. |
| `mipmap-ldpi/ic_launcher_round.png` | Android launcher ic launcher round artwork selected for mipmap-ldpi. |
| `mipmap-mdpi/ic_launcher.png` | Android launcher ic launcher artwork selected for mipmap-mdpi. |
| `mipmap-mdpi/ic_launcher_background.png` | Android launcher ic launcher background artwork selected for mipmap-mdpi. |
| `mipmap-mdpi/ic_launcher_foreground.png` | Android launcher ic launcher foreground artwork selected for mipmap-mdpi. |
| `mipmap-mdpi/ic_launcher_round.png` | Android launcher ic launcher round artwork selected for mipmap-mdpi. |
| `mipmap-xhdpi/ic_launcher.png` | Android launcher ic launcher artwork selected for mipmap-xhdpi. |
| `mipmap-xhdpi/ic_launcher_background.png` | Android launcher ic launcher background artwork selected for mipmap-xhdpi. |
| `mipmap-xhdpi/ic_launcher_foreground.png` | Android launcher ic launcher foreground artwork selected for mipmap-xhdpi. |
| `mipmap-xhdpi/ic_launcher_round.png` | Android launcher ic launcher round artwork selected for mipmap-xhdpi. |
| `mipmap-xxhdpi/ic_launcher.png` | Android launcher ic launcher artwork selected for mipmap-xxhdpi. |
| `mipmap-xxhdpi/ic_launcher_background.png` | Android launcher ic launcher background artwork selected for mipmap-xxhdpi. |
| `mipmap-xxhdpi/ic_launcher_foreground.png` | Android launcher ic launcher foreground artwork selected for mipmap-xxhdpi. |
| `mipmap-xxhdpi/ic_launcher_round.png` | Android launcher ic launcher round artwork selected for mipmap-xxhdpi. |
| `mipmap-xxxhdpi/ic_launcher.png` | Android launcher ic launcher artwork selected for mipmap-xxxhdpi. |
| `mipmap-xxxhdpi/ic_launcher_background.png` | Android launcher ic launcher background artwork selected for mipmap-xxxhdpi. |
| `mipmap-xxxhdpi/ic_launcher_foreground.png` | Android launcher ic launcher foreground artwork selected for mipmap-xxxhdpi. |
| `mipmap-xxxhdpi/ic_launcher_round.png` | Android launcher ic launcher round artwork selected for mipmap-xxxhdpi. |
| `values/colors.xml` | Android colors resource used by the Capacitor app. |
| `values/ic_launcher_background.xml` | Android ic launcher background resource used by the Capacitor app. |
| `values/strings.xml` | Android strings resource used by the Capacitor app. |
| `values/styles.xml` | Android styles resource used by the Capacitor app. |
| `xml/file_paths.xml` | Android file paths resource used by the Capacitor app. |
| `xml/network_security_config.xml` | Android network security config resource used by the Capacitor app. |

### `frontend/android/app/src/test/java/com/getcapacitor/myapp`

| File | Responsibility and connections |
|---|---|
| `ExampleUnitTest.java` | Template local JVM unit test for the Android project. |

### `frontend/android/gradle/wrapper`

| File | Responsibility and connections |
|---|---|
| `gradle-wrapper.jar` | Bootstraps the pinned Gradle distribution. |
| `gradle-wrapper.properties` | Selects the Gradle distribution used by both wrapper launchers. |

### `frontend/assets`

| File | Responsibility and connections |
|---|---|
| `icon.jpg` | Web brand icon source artwork. |

### `frontend/icons`

| File | Responsibility and connections |
|---|---|
| `icon-128.webp` | Sized web app icon for installation and launcher surfaces. |
| `icon-192.webp` | Sized web app icon for installation and launcher surfaces. |
| `icon-256.webp` | Sized web app icon for installation and launcher surfaces. |
| `icon-48.webp` | Sized web app icon for installation and launcher surfaces. |
| `icon-512.webp` | Sized web app icon for installation and launcher surfaces. |
| `icon-72.webp` | Sized web app icon for installation and launcher surfaces. |
| `icon-96.webp` | Sized web app icon for installation and launcher surfaces. |

### `frontend/netlify/functions-future`

| File | Responsibility and connections |
|---|---|
| `README.md` | Repository reference for frontend/netlify/functions-future setup and use. Compare its assumptions with the inspected code. |
| `gameweeks.js` | Staged future Netlify handler for calendar reads outside the deployed functions directory. Defines `handler`. |
| `users.js` | Staged future user/profile handler outside the deployed functions directory. Defines `handler`. |

### `frontend/netlify/functions`

| File | Responsibility and connections |
|---|---|
| `_supabase.js` | Builds the server-side database client and paginates reads past the row limit. Defines `jsonResponse`, `getSupabaseClient`, `selectAll`. Referenced by `fantasy.js`, `fixtures.js`. |
| `fantasy.js` | Implements the legacy web fantasy API using Supabase functions and the proposed ownership model. Defines `handler`. Uses `_supabase.js`. |
| `fixtures.js` | Builds the web fixture response with scores, kickoff and scorer enrichment. Defines `handler`. Uses `_supabase.js`. |
| `health.js` | Reports database/read health for the Netlify web API. Defines `handler`. Uses `_supabase.js`. |
| `matches.js` | Combines team sheets, events, scores and statistics for match detail. Defines `handler`. Uses `_supabase.js`. |
| `news.js` | Serves the web headline view from the worker’s sa_football_news cache. Defines `handler`. |
| `player-rankings.js` | Aggregates player performances for the web statistical leader lists. Defines `handler`. Uses `_supabase.js`. |
| `players.js` | Lists players, inferred club, prices, images and generated rating attributes. Defines `handler`. Uses `_supabase.js`. |
| `standings.js` | Builds the real-club league table for the web client. Defines `handler`. Uses `_supabase.js`. |

### `frontend/public/jerseys`

| File | Responsibility and connections |
|---|---|
| `Amzulu2.jpeg` | Web kit artwork for player and pitch views. |
| `Marumo.jpeg` | Web kit artwork for player and pitch views. |
| `Orbit college.jpeg` | Web kit artwork for player and pitch views. |
| `Ts galaxy.jpeg` | Web kit artwork for player and pitch views. |
| `amazulu-kit.svg` | Web kit artwork for player and pitch views. |
| `capetowncity-kit.svg` | Web kit artwork for player and pitch views. |
| `capetowncity.png` | Web kit artwork for player and pitch views. |
| `chiefs-kit.svg` | Web kit artwork for player and pitch views. |
| `chiefs.jpeg` | Web kit artwork for player and pitch views. |
| `chippa united.jpeg` | Web kit artwork for player and pitch views. |
| `chippa-kit.svg` | Web kit artwork for player and pitch views. |
| `durban city.jpeg` | Web kit artwork for player and pitch views. |
| `durbancity-kit.svg` | Web kit artwork for player and pitch views. |
| `gk-kit.svg` | Web kit artwork for player and pitch views. |
| `golden arrows.jpeg` | Web kit artwork for player and pitch views. |
| `goldenarrows-kit.svg` | Web kit artwork for player and pitch views. |
| `magesi-kit.svg` | Web kit artwork for player and pitch views. |
| `marumo-kit.svg` | Web kit artwork for player and pitch views. |
| `orlando pirates.webp` | Web kit artwork for player and pitch views. |
| `pirates-kit.svg` | Web kit artwork for player and pitch views. |
| `polokwane Fc.jpeg` | Web kit artwork for player and pitch views. |
| `polokwane-kit.svg` | Web kit artwork for player and pitch views. |
| `richards.png` | Web kit artwork for player and pitch views. |
| `richardsbay-kit.svg` | Web kit artwork for player and pitch views. |
| `sekhukhune-kit.svg` | Web kit artwork for player and pitch views. |
| `sekhukhune.jpeg` | Web kit artwork for player and pitch views. |
| `siwelele fc.jpeg` | Web kit artwork for player and pitch views. |
| `stellenbosch-kit.svg` | Web kit artwork for player and pitch views. |
| `stellenbosch.png` | Web kit artwork for player and pitch views. |
| `sundowns-kit.svg` | Web kit artwork for player and pitch views. |
| `sundowns.jpg` | Web kit artwork for player and pitch views. |
| `supersport-kit.svg` | Web kit artwork for player and pitch views. |
| `supersport.png` | Web kit artwork for player and pitch views. |
| `tsgalaxy-kit.svg` | Web kit artwork for player and pitch views. |

### `frontend/public/logos`

| File | Responsibility and connections |
|---|---|
| `AmaZulu FC logo.png` | Web image in the club-crest collection. |
| `Chippa United logo.png` | Web image in the club-crest collection. |
| `Durban City FC logo.png` | Web image in the club-crest collection. |
| `Golden Arrows logo.png` | Web image in the club-crest collection. |
| `Kaizer Chiefs logo.png` | Web image in the club-crest collection. |
| `Magesi FC logo.png` | Web image in the club-crest collection. |
| `Mamelodi Sundowns logo.png` | Web image in the club-crest collection. |
| `Marumo Gallants logo.png` | Web image in the club-crest collection. |
| `Orbit College logo.png` | Web image in the club-crest collection. |
| `Orland Pirates logo.png` | Web image in the club-crest collection. |
| `Polokwane City logo.png` | Web image in the club-crest collection. |
| `Richards Bay logo.png` | Web image in the club-crest collection. |
| `Screenshot 2026-01-13 214028.png` | Web image in the club-crest collection. |
| `Screenshot 2026-01-13 214317.png` | Web image in the club-crest collection. |
| `Screenshot 2026-01-13 214352.png` | Web image in the club-crest collection. |
| `Screenshot 2026-01-13 214423.png` | Web image in the club-crest collection. |
| `Screenshot 2026-01-13 214443.png` | Web image in the club-crest collection. |
| `Screenshot 2026-01-13 214519.png` | Web image in the club-crest collection. |
| `Screenshot 2026-01-13 214544.png` | Web image in the club-crest collection. |
| `Screenshot 2026-01-13 214658.png` | Web image in the club-crest collection. |
| `Screenshot 2026-01-13 214717.png` | Web image in the club-crest collection. |
| `Screenshot 2026-01-13 214812.png` | Web image in the club-crest collection. |
| `Screenshot 2026-01-13 214839.png` | Web image in the club-crest collection. |
| `Screenshot 2026-01-13 214922.png` | Web image in the club-crest collection. |
| `Screenshot 2026-01-13 214953.png` | Web image in the club-crest collection. |
| `Screenshot 2026-01-13 215015.png` | Web image in the club-crest collection. |
| `Screenshot 2026-01-13 215041.png` | Web image in the club-crest collection. |
| `Screenshot 2026-01-13 215101.png` | Web image in the club-crest collection. |
| `Sekhukhune United FC logo.png` | Web image in the club-crest collection. |
| `Siwelele FC logo.png` | Web image in the club-crest collection. |
| `Stellenbosch FC logo.png` | Web image in the club-crest collection. |
| `TS galaxy logo.png` | Web image in the club-crest collection. |

### `frontend/public`

| File | Responsibility and connections |
|---|---|
| `manifest.webmanifest` | Declares the installable web app’s display, name and icons. |
| `placeholder.svg` | Fallback illustration for missing web imagery. |
| `privacy.html` | Public privacy page served by the web site’s privacy redirect. |
| `robots.txt` | Crawler directives for the public web site. |

### `frontend/scripts`

| File | Responsibility and connections |
|---|---|
| `dev-api-mock.mjs` | Serves a news fixture locally and proxies other API reads for UI development. |
| `dev-api.mjs` | Runs the actual Netlify HTTP handlers locally against configured Supabase data. |
| `fetch-stat-types.mjs` | Resolves provider statistic codes so match labels can be verified. |
| `generate-jerseys.mjs` | Generates static shirt artwork for the football components. |

### `frontend/src`

| File | Responsibility and connections |
|---|---|
| `App.css` | Styles scoped to the root web application. |
| `App.tsx` | Mounts routing, providers, notifications and the PWA prompt. Uses `toaster.tsx`, `sonner.tsx`, `tooltip.tsx`. |
| `index.css` | Global web styles, Tailwind layers and shared variables. Referenced by `main.tsx`. |
| `main.tsx` | Mounts the React web root and loads global styles. Uses `App.tsx`, `index.css`, `ErrorBoundary.tsx`. |
| `vite-env.d.ts` | Vite environment type declarations for the web build. |

### `frontend/src/assets/jerseys`

| File | Responsibility and connections |
|---|---|
| `cape-town-city-jersey.png` | Web kit artwork for player and pitch views. |
| `chiefs-jersey.jpg` | Web kit artwork for player and pitch views. |
| `chiefs-jersey.png` | Web kit artwork for player and pitch views. |
| `default-jersey.png` | Web kit artwork for player and pitch views. |
| `pirates-2026.jpg` | Web kit artwork for player and pitch views. |
| `pirates-jersey.png` | Web kit artwork for player and pitch views. |
| `sundowns-jersey.png` | Web kit artwork for player and pitch views. |
| `supersport-jersey.png` | Web kit artwork for player and pitch views. |

### `frontend/src/components`

| File | Responsibility and connections |
|---|---|
| `ErrorBoundary.tsx` | Provides a fallback if a descendant React view throws during rendering. Defines `ErrorBoundary`. Uses `button.tsx`, `card.tsx`. |
| `PwaInstallPrompt.tsx` | Presents install guidance for the web application. Defines `PwaInstallPrompt`. Uses `button.tsx`, `card.tsx`. |

### `frontend/src/components/football`

| File | Responsibility and connections |
|---|---|
| `AuthModal.tsx` | Browser account forms wired to the web auth context. Defines `AuthModal`. Uses `AuthContext.tsx`, `pslClubs.ts`, `TeamCrest.tsx`. |
| `FixtureRow.tsx` | Displays a fixture’s clubs, kickoff, state and score in a list. Defines `FixtureRow`. Uses `touchlineApi.ts`, `TeamCrest.tsx`, `icons.tsx`. |
| `FootballDashboard.tsx` | Coordinates football tabs and connects list, club and match views. Defines `FootballDashboard`. Uses `icons.tsx`, `button.tsx`, `dialog.tsx`. |
| `LiveMatchTimer.tsx` | Derives a display clock for an in-progress fixture. Defines `LiveTimerInfo`, `calculateLiveMatchTime`, `useLiveMatchTimer`. Uses `touchlineApi.ts`, `utils.ts`. |
| `MatchDetail.tsx` | Displays selected match events, lineups and team statistics. Defines `MatchDetail`. Uses `dialog.tsx`, `tabs.tsx`, `touchlineApi.ts`. |
| `MyClubHub.tsx` | Filters football information for the fan’s chosen club. Defines `MyClubHub`. Uses `AuthContext.tsx`, `ClubThemeContext.tsx`, `touchlineApi.ts`. |
| `NewsList.tsx` | Loads and renders headlines and publisher links in the web layout. Defines `NewsList`. Uses `touchlineApi.ts`, `icons.tsx`. |
| `Pitch.tsx` | Renders a football pitch and match lineup positions. Defines `Pitch`. Uses `touchlineApi.ts`, `TeamCrest.tsx`, `icons.tsx`. |
| `StandingsTable.tsx` | Displays club rankings, results and table statistics. Defines `StandingsTable`. Uses `touchlineApi.ts`, `TeamCrest.tsx`, `utils.ts`. |
| `TeamCrest.tsx` | Resolves or substitutes artwork for a club crest. Defines `TeamCrest`. Referenced by `AuthModal.tsx`, `FixtureRow.tsx`. |
| `TeamJersey.tsx` | Displays a team’s kit artwork for football views. Defines `TeamJerseyProps`, `TeamJersey`. Uses `pslClubs.ts`, `auth.ts`. |
| `icons.tsx` | Defines the football UI’s shared web icons. Defines `ICON`, `CardMark`. Referenced by `FixtureRow.tsx`, `FootballDashboard.tsx`. |

### `frontend/src/components/ui`

| File | Responsibility and connections |
|---|---|
| `accordion.tsx` | Web accordion primitive for composing React screens. Uses `utils.ts`. |
| `alert-dialog.tsx` | Web alert dialog primitive for composing React screens. Uses `utils.ts`, `button.tsx`. |
| `alert.tsx` | Web alert primitive for composing React screens. Uses `utils.ts`. |
| `aspect-ratio.tsx` | Web aspect ratio primitive for composing React screens. |
| `avatar.tsx` | Web avatar primitive for composing React screens. Uses `utils.ts`. |
| `badge.tsx` | Web badge primitive for composing React screens. Uses `utils.ts`. |
| `breadcrumb.tsx` | Web breadcrumb primitive for composing React screens. Uses `utils.ts`. |
| `button.tsx` | Web button primitive for composing React screens. Uses `utils.ts`. |
| `calendar.tsx` | Web calendar primitive for composing React screens. Uses `utils.ts`, `button.tsx`. |
| `card.tsx` | Web card primitive for composing React screens. Uses `utils.ts`. |
| `carousel.tsx` | Web carousel primitive for composing React screens. Uses `utils.ts`, `button.tsx`. |
| `chart.tsx` | Web chart primitive for composing React screens. Uses `utils.ts`. |
| `checkbox.tsx` | Web checkbox primitive for composing React screens. Uses `utils.ts`. |
| `collapsible.tsx` | Web collapsible primitive for composing React screens. |
| `command.tsx` | Web command primitive for composing React screens. Uses `utils.ts`, `dialog.tsx`. |
| `context-menu.tsx` | Web context menu primitive for composing React screens. Uses `utils.ts`. |
| `dialog.tsx` | Web dialog primitive for composing React screens. Uses `utils.ts`. |
| `drawer.tsx` | Web drawer primitive for composing React screens. Uses `utils.ts`. |
| `dropdown-menu.tsx` | Web dropdown menu primitive for composing React screens. Uses `utils.ts`. |
| `form.tsx` | Web form primitive for composing React screens. Uses `utils.ts`, `label.tsx`. |
| `hover-card.tsx` | Web hover card primitive for composing React screens. Uses `utils.ts`. |
| `input-otp.tsx` | Web input otp primitive for composing React screens. Uses `utils.ts`. |
| `input.tsx` | Web input primitive for composing React screens. Uses `utils.ts`. |
| `label.tsx` | Web label primitive for composing React screens. Uses `utils.ts`. |
| `menubar.tsx` | Web menubar primitive for composing React screens. Uses `utils.ts`. |
| `navigation-menu.tsx` | Web navigation menu primitive for composing React screens. Uses `utils.ts`. |
| `pagination.tsx` | Web pagination primitive for composing React screens. Uses `utils.ts`, `button.tsx`. |
| `popover.tsx` | Web popover primitive for composing React screens. Uses `utils.ts`. |
| `progress.tsx` | Web progress primitive for composing React screens. Uses `utils.ts`. |
| `radio-group.tsx` | Web radio group primitive for composing React screens. Uses `utils.ts`. |
| `resizable.tsx` | Web resizable primitive for composing React screens. Uses `utils.ts`. |
| `scroll-area.tsx` | Web scroll area primitive for composing React screens. Uses `utils.ts`. |
| `select.tsx` | Web select primitive for composing React screens. Uses `utils.ts`. |
| `separator.tsx` | Web separator primitive for composing React screens. Uses `utils.ts`. |
| `sheet.tsx` | Web sheet primitive for composing React screens. Uses `utils.ts`. |
| `sidebar.tsx` | Web sidebar primitive for composing React screens. Uses `use-mobile.tsx`, `utils.ts`, `button.tsx`. |
| `skeleton.tsx` | Web skeleton primitive for composing React screens. Uses `utils.ts`. |
| `slider.tsx` | Web slider primitive for composing React screens. Uses `utils.ts`. |
| `sonner.tsx` | Web sonner primitive for composing React screens. Referenced by `App.tsx`. |
| `switch.tsx` | Web switch primitive for composing React screens. Uses `utils.ts`. |
| `table.tsx` | Loads football data and displays the table derived from finished fixtures. Uses `utils.ts`. |
| `tabs.tsx` | Web tabs primitive for composing React screens. Uses `utils.ts`. |
| `textarea.tsx` | Web textarea primitive for composing React screens. Uses `utils.ts`. |
| `toast.tsx` | Web toast primitive for composing React screens. Uses `utils.ts`. |
| `toaster.tsx` | Web toaster primitive for composing React screens. Uses `use-toast.ts`, `toast.tsx`. |
| `toggle-group.tsx` | Web toggle group primitive for composing React screens. Uses `utils.ts`, `toggle.tsx`. |
| `toggle.tsx` | Web toggle primitive for composing React screens. Uses `utils.ts`. |
| `tooltip.tsx` | Web tooltip primitive for composing React screens. Uses `utils.ts`. |
| `use-toast.ts` | Manages toast state and notifications for the web component system. Uses `use-toast.ts`. |

### `frontend/src/constants`

| File | Responsibility and connections |
|---|---|
| `pslClubs.ts` | PSL club catalogue used for favourite-club selection and display. Defines `PSL_CLUBS`, `GK_JERSEY`, `getPslClubById`. Uses `auth.ts`. |

### `frontend/src/context`

| File | Responsibility and connections |
|---|---|
| `AuthContext.tsx` | Publishes browser-local auth operations and session state to the dashboard. Defines `AuthProvider`, `useAuth`. Uses `auth.ts`, `authService.ts`. |
| `ClubThemeContext.tsx` | Supplies club-dependent visual choices to web components. Defines `ClubThemeProvider`, `useClubTheme`. Uses `AuthContext.tsx`, `auth.ts`, `pslClubs.ts`. |

### `frontend/src/hooks`

| File | Responsibility and connections |
|---|---|
| `use-mobile.tsx` | Reports responsive mobile breakpoint state for the web UI. Defines `useIsMobile`. Referenced by `sidebar.tsx`. |
| `use-toast.ts` | Manages toast state and notifications for the web component system. Defines `reducer`. Uses `toast.tsx`. |

### `frontend/src/lib`

| File | Responsibility and connections |
|---|---|
| `utils.ts` | Combines conditional CSS classes for web components. Defines `cn`. Referenced by `AuthModal.tsx`, `FixtureRow.tsx`. |

### `frontend/src/pages`

| File | Responsibility and connections |
|---|---|
| `Index.tsx` | Root web page that renders the football dashboard. Uses `FootballDashboard.tsx`. |
| `NotFound.tsx` | Fallback page for an unknown browser route. Referenced by `App.tsx`. |

### `frontend/src/services`

| File | Responsibility and connections |
|---|---|
| `authService.ts` | Persists the browser-local account registry and current session. Defines `getCurrentUser`, `saveSession`, `clearSession`. Uses `auth.ts`, `pslClubs.ts`. |
| `fantasyRules.ts` | Browser-side chip, transfer and private league rule helpers. Defines `MINI_LEAGUE_CAPACITY`, `RuleResult`, `activateChip`. Uses `fantasy.ts`. |
| `fantasyScoring.ts` | Browser-side calculations for player and squad scores. Defines `calculatePlayerScore`, `calculateSquadTotal`. Uses `fantasy.ts`. |
| `fantasySquadService.ts` | Stores per-browser squad drafts and validates local budget and selection. Defines `INITIAL_BUDGET`, `MAX_PLAYERS_PER_CLUB`, `ROSTER_REQUIREMENTS`. Uses `touchlineApi.ts`, `fantasy.ts`. |
| `touchlineApi.ts` | Typed web football/news client and API-base selection for browser or Capacitor. Defines `Position`, `ApiPlayer`, `FixtureStatus`. Referenced by `FixtureRow.tsx`, `FootballDashboard.tsx`. |

### `frontend/src/types`

| File | Responsibility and connections |
|---|---|
| `auth.ts` | Browser identity and registration payload types. Defines `PslClub`, `User`, `AuthState`. Referenced by `TeamJersey.tsx`, `pslClubs.ts`. |
| `fantasy.ts` | Web fantasy squad, league, chip and player model types. Defines `Formation`, `FormationLayout`, `FORMATIONS`. Uses `touchlineApi.ts`. |

### `frontend/supabase`

| File | Responsibility and connections |
|---|---|
| `config.toml` | Supabase CLI project settings for the older web integration. |

### `frontend/supabase/functions/migrate-local-data`

| File | Responsibility and connections |
|---|---|
| `index.ts` | Legacy edge function for importing locally held application data into Supabase. |

### `frontend/supabase/functions/populate-database`

| File | Responsibility and connections |
|---|---|
| `index.ts` | Legacy edge function that populates the older Supabase application model. |

### `frontend/supabase/migrations`

| File | Responsibility and connections |
|---|---|
| `20250805151037_188eff21-0a26-4ac6-bb6e-b990134f2ced.sql` | Legacy Supabase migration. Enable Row Level Security on all tables. |
| `20250805151108_aee453d6-16d9-458a-b146-112cb2abeffb.sql` | Legacy Supabase migration. Enable RLS on Fantasy league table. |
| `20251028113335_1c08574f-71f5-472b-ac3a-f7e7a9112d42.sql` | Legacy Supabase migration. Add unique constraint to team_name in profiles table. |
| `20251030002711_034c08e0-9845-4870-b3f7-e80ae4752a6d.sql` | Legacy Supabase migration. Add policies for regular users to view players (needed to build teams). |
| `20251030005756_a73102d2-114e-4cc5-b65e-cf7fb1787378.sql` | Legacy Supabase migration. Create policy to allow users to rename teams after 36 weeks. |
| `20251031094536_e566a11d-2de3-498a-acbe-03a1fbc54ff5.sql` | Legacy Supabase migration. Create app_role enum if it doesn't exist. |

### `mobile-app`

| File | Responsibility and connections |
|---|---|
| `.env.example` | Lists environment setting names and formats. Supply actual values privately per environment. |
| `.gitignore` | Git exclusions for local settings and generated files. |
| `AGENTS.md` | Repository reference for agents. Compare its assumptions with the inspected code. |
| `IMPLEMENTATION_GUIDE.md` | Repository reference for implementation guide. Compare its assumptions with the inspected code. |
| `LICENSE` | Licence accompanying the native app project. |
| `app.json` | Expo app identity, bundle/package identifiers, scheme, orientation, icons and plugins. |
| `figma-leagues.png` | Native leagues design reference for comparing the rendered screen. |
| `figma-live.png` | Native live design reference for comparing the rendered screen. |
| `figma-news.png` | Native news design reference for comparing the rendered screen. |
| `figma-squad.png` | Native squad design reference for comparing the rendered screen. |
| `figma-table.png` | Native table design reference for comparing the rendered screen. |
| `package-lock.json` | Pins npm package resolutions for reproducible installs. |
| `package.json` | Declares libraries and runnable scripts for mobile-app. |
| `tsconfig.json` | TypeScript compile scope, target and resolution settings for this build. |

### `mobile-app/.vscode`

| File | Responsibility and connections |
|---|---|
| `extensions.json` | Suggested editor extensions for the mobile project. |
| `settings.json` | Mobile project workspace editor settings. |

### `mobile-app/app/(tabs)`

| File | Responsibility and connections |
|---|---|
| `_layout.tsx` | Declares five visible product tabs and hides the starter route. Defines `TabLayout`. Uses `design.ts`. |
| `index.tsx` | Native News screen with category filtering, refresh and publisher links. Defines `NewsScreen`. Uses `TouchlineHeader.tsx`, `ScreenState.tsx`, `client.ts`. |
| `leagues.tsx` | Native Leagues tab with the coming-soon state. Defines `LeaguesScreen`. Uses `design.ts`. |
| `live.tsx` | Native fixtures/teams loader and grouped live games display with pull-to-refresh. Defines `LiveGamesScreen`. Uses `client.ts`, `useApi.ts`, `types.ts`. |
| `squad.tsx` | Native manager dashboard, team creation, snapshot presentation and searchable market. Defines `MySquadScreen`. Uses `client.ts`, `useApi.ts`, `types.ts`. |
| `table.tsx` | Loads football data and displays the table derived from finished fixtures. Defines `TableScreen`. Uses `client.ts`, `useApi.ts`, `ScreenState.tsx`. |
| `two.tsx` | Starter route kept hidden from the five product tabs. Defines `TabTwoScreen`. Uses `EditScreenInfo.tsx`, `Themed.tsx`. |

### `mobile-app/app`

| File | Responsibility and connections |
|---|---|
| `+html.tsx` | Defines the HTML wrapper when rendering Expo Router for web. Defines `Root`. |
| `+not-found.tsx` | Handles unmatched native-router paths. Defines `NotFoundScreen`. Uses `Themed.tsx`. |
| `_layout.tsx` | Loads fonts, coordinates splash dismissal and wraps routes in the auth provider. Defines `unstable_settings`, `RootLayout`. Uses `AuthContext.tsx`, `SpaceMono-Regular.ttf`. |
| `modal.tsx` | Account modal for registration, sign-in and sign-out with platform-aware social options. Defines `StartScreen`. Uses `AuthContext.tsx`, `supabaseClient.ts`, `design.ts`. |

### `mobile-app/assets/fonts`

| File | Responsibility and connections |
|---|---|
| `SpaceMono-Regular.ttf` | Monospace font bundled and loaded by the root native layout. Referenced by `_layout.tsx`. |

### `mobile-app/assets/images`

| File | Responsibility and connections |
|---|---|
| `android-icon-background.png` | Expo android icon background artwork for native branding or web export. |
| `android-icon-foreground.png` | Expo android icon foreground artwork for native branding or web export. |
| `android-icon-monochrome.png` | Expo android icon monochrome artwork for native branding or web export. |
| `favicon.png` | Expo favicon artwork for native branding or web export. |
| `icon.png` | Expo icon artwork for native branding or web export. |
| `splash-icon.png` | Expo splash icon artwork for native branding or web export. |

### `mobile-app/assets/jerseys`

| File | Responsibility and connections |
|---|---|
| `Amzulu2.jpeg` | Bundled native kit artwork for player and pitch views. |
| `Marumo.jpeg` | Bundled native kit artwork for player and pitch views. |
| `Orbit college.jpeg` | Bundled native kit artwork for player and pitch views. |
| `Ts galaxy.jpeg` | Bundled native kit artwork for player and pitch views. |
| `amazulu-kit.svg` | Bundled native kit artwork for player and pitch views. |
| `capetowncity-kit.svg` | Bundled native kit artwork for player and pitch views. |
| `capetowncity.png` | Bundled native kit artwork for player and pitch views. |
| `chiefs-kit.svg` | Bundled native kit artwork for player and pitch views. |
| `chiefs.jpeg` | Bundled native kit artwork for player and pitch views. |
| `chippa united.jpeg` | Bundled native kit artwork for player and pitch views. |
| `chippa-kit.svg` | Bundled native kit artwork for player and pitch views. |
| `durban city.jpeg` | Bundled native kit artwork for player and pitch views. |
| `durbancity-kit.svg` | Bundled native kit artwork for player and pitch views. |
| `gk-kit.svg` | Bundled native kit artwork for player and pitch views. |
| `golden arrows.jpeg` | Bundled native kit artwork for player and pitch views. |
| `goldenarrows-kit.svg` | Bundled native kit artwork for player and pitch views. |
| `magesi-kit.svg` | Bundled native kit artwork for player and pitch views. |
| `marumo-kit.svg` | Bundled native kit artwork for player and pitch views. |
| `orlando pirates.webp` | Bundled native kit artwork for player and pitch views. |
| `pirates-kit.svg` | Bundled native kit artwork for player and pitch views. |
| `polokwane Fc.jpeg` | Bundled native kit artwork for player and pitch views. |
| `polokwane-kit.svg` | Bundled native kit artwork for player and pitch views. |
| `richards.png` | Bundled native kit artwork for player and pitch views. |
| `richardsbay-kit.svg` | Bundled native kit artwork for player and pitch views. |
| `sekhukhune-kit.svg` | Bundled native kit artwork for player and pitch views. |
| `sekhukhune.jpeg` | Bundled native kit artwork for player and pitch views. |
| `siwelele fc.jpeg` | Bundled native kit artwork for player and pitch views. |
| `stellenbosch-kit.svg` | Bundled native kit artwork for player and pitch views. |
| `stellenbosch.png` | Bundled native kit artwork for player and pitch views. |
| `sundowns-kit.svg` | Bundled native kit artwork for player and pitch views. |
| `sundowns.jpg` | Bundled native kit artwork for player and pitch views. |
| `supersport-kit.svg` | Bundled native kit artwork for player and pitch views. |
| `supersport.png` | Bundled native kit artwork for player and pitch views. |
| `tsgalaxy-kit.svg` | Bundled native kit artwork for player and pitch views. |

### `mobile-app/assets/logos`

| File | Responsibility and connections |
|---|---|
| `AmaZulu FC logo.png` | Bundled native image in the club-crest collection. Referenced by `crests.ts`. |
| `Chippa United logo.png` | Bundled native image in the club-crest collection. Referenced by `crests.ts`. |
| `Durban City FC logo.png` | Bundled native image in the club-crest collection. Referenced by `crests.ts`. |
| `Golden Arrows logo.png` | Bundled native image in the club-crest collection. Referenced by `crests.ts`. |
| `Kaizer Chiefs logo.png` | Bundled native image in the club-crest collection. Referenced by `crests.ts`. |
| `Magesi FC logo.png` | Bundled native image in the club-crest collection. Referenced by `crests.ts`. |
| `Mamelodi Sundowns logo.png` | Bundled native image in the club-crest collection. Referenced by `crests.ts`. |
| `Marumo Gallants logo.png` | Bundled native image in the club-crest collection. Referenced by `crests.ts`. |
| `Orbit College logo.png` | Bundled native image in the club-crest collection. Referenced by `crests.ts`. |
| `Orland Pirates logo.png` | Bundled native image in the club-crest collection. Referenced by `crests.ts`. |
| `Polokwane City logo.png` | Bundled native image in the club-crest collection. Referenced by `crests.ts`. |
| `Richards Bay logo.png` | Bundled native image in the club-crest collection. Referenced by `crests.ts`. |
| `Screenshot 2026-01-13 214028.png` | Bundled native image in the club-crest collection. |
| `Screenshot 2026-01-13 214317.png` | Bundled native image in the club-crest collection. |
| `Screenshot 2026-01-13 214352.png` | Bundled native image in the club-crest collection. |
| `Screenshot 2026-01-13 214423.png` | Bundled native image in the club-crest collection. |
| `Screenshot 2026-01-13 214443.png` | Bundled native image in the club-crest collection. |
| `Screenshot 2026-01-13 214519.png` | Bundled native image in the club-crest collection. |
| `Screenshot 2026-01-13 214544.png` | Bundled native image in the club-crest collection. |
| `Screenshot 2026-01-13 214658.png` | Bundled native image in the club-crest collection. |
| `Screenshot 2026-01-13 214717.png` | Bundled native image in the club-crest collection. |
| `Screenshot 2026-01-13 214812.png` | Bundled native image in the club-crest collection. |
| `Screenshot 2026-01-13 214839.png` | Bundled native image in the club-crest collection. |
| `Screenshot 2026-01-13 214922.png` | Bundled native image in the club-crest collection. |
| `Screenshot 2026-01-13 214953.png` | Bundled native image in the club-crest collection. |
| `Screenshot 2026-01-13 215015.png` | Bundled native image in the club-crest collection. |
| `Screenshot 2026-01-13 215041.png` | Bundled native image in the club-crest collection. |
| `Screenshot 2026-01-13 215101.png` | Bundled native image in the club-crest collection. |
| `Sekhukhune United FC logo.png` | Bundled native image in the club-crest collection. Referenced by `crests.ts`. |
| `Siwelele FC logo.png` | Bundled native image in the club-crest collection. Referenced by `crests.ts`. |
| `Stellenbosch FC logo.png` | Bundled native image in the club-crest collection. Referenced by `crests.ts`. |
| `TS galaxy logo.png` | Bundled native image in the club-crest collection. Referenced by `crests.ts`. |

### `mobile-app/components`

| File | Responsibility and connections |
|---|---|
| `EditScreenInfo.tsx` | Starter component providing template screen editing information. Defines `EditScreenInfo`. Uses `ExternalLink.tsx`, `StyledText.tsx`, `Themed.tsx`. |
| `ExternalLink.tsx` | Wraps external links with platform-aware navigation. Defines `ExternalLink`. Referenced by `EditScreenInfo.tsx`. |
| `StyledText.tsx` | Reusable monospace text wrapper in the starter UI. Defines `MonoText`. Uses `Themed.tsx`. |
| `Themed.tsx` | Starter theme-aware wrappers for text and views. Defines `TextProps`, `ViewProps`, `useThemeColor`. Uses `useColorScheme.ts`, `Colors.ts`. |
| `useClientOnlyValue.ts` | Supplies the client value for native platform rendering. Defines `useClientOnlyValue`. |
| `useClientOnlyValue.web.ts` | Defers client-only values around web hydration. Defines `useClientOnlyValue`. |
| `useColorScheme.ts` | Native colour-scheme hook for theme-aware components. Defines `useColorScheme`. Referenced by `Themed.tsx`. |
| `useColorScheme.web.ts` | Web-specific colour-scheme hook for rendered Expo screens. Defines `useColorScheme`. |

### `mobile-app/constants`

| File | Responsibility and connections |
|---|---|
| `Colors.ts` | Starter light and dark palette for the theme-aware components. Referenced by `EditScreenInfo.tsx`, `Themed.tsx`. |

### `mobile-app/src/components`

| File | Responsibility and connections |
|---|---|
| `Common.tsx` | Reusable title pill and crest display used in native football screens. Defines `TitlePill`, `Crest`. Uses `design.ts`, `crests.ts`. |
| `ScreenState.tsx` | Reusable loading, error, empty and recovery presentation. Defines `ScreenState`. Uses `index.ts`, `design.ts`. |
| `TouchlineHeader.tsx` | Shared mobile header with branding and access to account controls. Defines `TouchlineHeader`. Uses `design.ts`. |

### `mobile-app/src/lib/api`

| File | Responsibility and connections |
|---|---|
| `client.ts` | Builds mobile HTTP requests, selects a base URL, attaches auth and maps errors. Defines `ApiError`, `api`. Uses `env.ts`, `supabaseClient.ts`, `types.ts`. |
| `types.ts` | Mobile transport types for football, fantasy, news and calculated standings. Defines `Position`, `MatchStatus`, `SeasonRef`. Referenced by `live.tsx`, `squad.tsx`. |
| `useApi.ts` | Runs an asynchronous screen loader and tracks newest request, retry and error state. Defines `useApi`. Referenced by `index.tsx`, `live.tsx`. |

### `mobile-app/src/lib/auth`

| File | Responsibility and connections |
|---|---|
| `AuthContext.tsx` | Restores the Supabase session, observes auth changes and exposes sign-in, sign-up and sign-out. Defines `AuthProvider`, `useAuth`. Uses `supabaseClient.ts`. |
| `secureStorage.ts` | Supabase session storage adapter for native SecureStore and web local storage. Defines `SecureStorageAdapter`. Referenced by `supabaseClient.ts`. |
| `supabaseClient.ts` | Creates the persistent, auto-refreshing mobile Auth client. Defines `supabase`. Uses `env.ts`, `secureStorage.ts`. |

### `mobile-app/src/lib/config`

| File | Responsibility and connections |
|---|---|
| `env.ts` | Selects platform API, news and Supabase origins plus request timeout. Defines `AppConfig`. Referenced by `client.ts`, `supabaseClient.ts`. |

### `mobile-app/src/lib`

| File | Responsibility and connections |
|---|---|
| `crests.ts` | Maps club names/assets for native crest fallback. Defines `crestSource`. Uses `AmaZulu FC logo.png`, `Chippa United logo.png`, `Durban City FC logo.png`. |
| `football.ts` | Builds team lookups and computes a real-club table from finished results. Defines `TeamLookup`, `buildTeamLookup`, `computeStandings`. Uses `types.ts`. |

### `mobile-app/src/theme`

| File | Responsibility and connections |
|---|---|
| `design.ts` | Figma-derived brand palette, display/body fonts and shadow settings for native screens. Defines `Brand`, `Fonts`, `Shadow`. Referenced by `_layout.tsx`, `index.tsx`. |
| `index.ts` | Alternative dark theme token set alongside the screen design tokens. Defines `Colors`, `Spacing`, `Typography`. Referenced by `ScreenState.tsx`. |

### `touchline-news-api`

| File | Responsibility and connections |
|---|---|
| `.env.example` | Lists environment setting names and formats. Supply actual values privately per environment. |
| `.gitignore` | Git exclusions for local settings and generated files. |
| `Dockerfile` | Image build and launch instructions for the Node news service. |
| `README.md` | Repository reference for touchline-news-api setup and use. Compare its assumptions with the inspected code. |
| `docker-compose.yml` | Local container services for running the news stack. |
| `package-lock.json` | Pins npm package resolutions for reproducible installs. |
| `package.json` | Declares libraries and runnable scripts for touchline-news-api. |
| `tsconfig.json` | TypeScript compile scope, target and resolution settings for this build. |
| `vitest.config.ts` | Node news test runner configuration. |

### `touchline-news-api/prisma/migrations/20260827000000_init`

| File | Responsibility and connections |
|---|---|
| `migration.sql` | Initial Prisma migration creating article, source, category, team tagging and sync log storage. |

### `touchline-news-api/prisma`

| File | Responsibility and connections |
|---|---|
| `schema.prisma` | Defines news entities, relations, uniqueness constraints and PostgreSQL datasource. |

### `touchline-news-api/scripts`

| File | Responsibility and connections |
|---|---|
| `seed.ts` | Populates the news service’s reference data. Uses `categorization.service.ts`, `deduplicate.ts`. |
| `sync-news.ts` | Runs a manual Node provider ingestion through the news sync service. Uses `database.ts`, `gnews.provider.ts`, `news-sync.service.ts`. |

### `touchline-news-api/src`

| File | Responsibility and connections |
|---|---|
| `app.ts` | Composes Express middleware, public routes, documentation and error handling. Defines `createApp`. Uses `database.ts`, `env.ts`, `error.middleware.ts`. |
| `server.ts` | Starts the Express listener and optional scheduled GNews import. Uses `database.ts`, `env.ts`, `app.ts`. |

### `touchline-news-api/src/config`

| File | Responsibility and connections |
|---|---|
| `database.ts` | Creates the Prisma database client for news queries and writes. Defines `prisma`. Referenced by `sync-news.ts`, `app.ts`. |
| `env.ts` | Validates Node runtime settings, provider configuration and scheduling options. Defines `env`, `newsQueries`. Referenced by `app.ts`, `rate-limit.middleware.ts`. |

### `touchline-news-api/src/controllers`

| File | Responsibility and connections |
|---|---|
| `categories.controller.ts` | Handles categories HTTP requests and shapes Prisma-backed news data. Defines `CategoriesController`. Referenced by `categories.routes.ts`. |
| `news.controller.ts` | Handles news HTTP requests and shapes Prisma-backed news data. Defines `NewsController`. Uses `news.service.ts`, `error.middleware.ts`. |
| `sources.controller.ts` | Handles sources HTTP requests and shapes Prisma-backed news data. Defines `SourcesController`. Referenced by `sources.routes.ts`. |
| `teams.controller.ts` | Handles teams HTTP requests and shapes Prisma-backed news data. Defines `TeamsController`. Uses `error.middleware.ts`, `news.service.ts`. |

### `touchline-news-api/src/middleware`

| File | Responsibility and connections |
|---|---|
| `error.middleware.ts` | Maps service/validation failures and unknown routes to responses. Defines `AppError`, `notFoundHandler`, `errorHandler`. Uses `logger.ts`. |
| `rate-limit.middleware.ts` | Applies request limits to Express API routes. Defines `apiRateLimiter`. Uses `env.ts`. |

### `touchline-news-api/src/providers`

| File | Responsibility and connections |
|---|---|
| `gnews.provider.ts` | Fetches and adapts GNews stories for ingestion. Defines `GNewsProvider`. Uses `env.ts`, `news.ts`, `news.provider.ts`. |
| `news.provider.ts` | Declares the news ingestion provider contract. Defines `NewsProvider`. Uses `news.ts`. |

### `touchline-news-api/src/routes`

| File | Responsibility and connections |
|---|---|
| `async-handler.ts` | Forwards rejected asynchronous Express handlers to error middleware. Defines `asyncHandler`. Referenced by `categories.routes.ts`, `news.routes.ts`. |
| `categories.routes.ts` | Connects categories paths to their Express controller. Defines `createCategoriesRouter`. Uses `categories.controller.ts`, `async-handler.ts`. |
| `news.routes.ts` | Connects news paths to their Express controller. Defines `createNewsRouter`. Uses `news.controller.ts`, `async-handler.ts`. |
| `sources.routes.ts` | Connects sources paths to their Express controller. Defines `createSourcesRouter`. Uses `sources.controller.ts`, `async-handler.ts`. |
| `teams.routes.ts` | Connects teams paths to their Express controller. Defines `createTeamsRouter`. Uses `teams.controller.ts`, `async-handler.ts`. |

### `touchline-news-api/src/services`

| File | Responsibility and connections |
|---|---|
| `categorization.service.ts` | Assigns category labels to stories using their content. Defines `CATEGORIES`, `CategorizationService`. Referenced by `seed.ts`, `news-sync.service.ts`. |
| `news-sync.service.ts` | Coordinates provider retrieval, duplicate checks, tagging, persistence and run counts. Defines `SyncSummary`, `NewsSyncService`. Uses `news.provider.ts`, `news.ts`, `deduplicate.ts`. |
| `news.service.ts` | Validates filters and serves paginated, searched and team/category article reads. Defines `listNewsQuerySchema`, `searchQuerySchema`, `NewsService`. Uses `news.ts`. |
| `team-detection.service.ts` | Detects club mentions using team names and aliases. Defines `DetectedTeam`, `TeamDetectionService`. Referenced by `news-sync.service.ts`, `team-detection.test.ts`. |

### `touchline-news-api/src/types`

| File | Responsibility and connections |
|---|---|
| `news.ts` | Contracts for provider inputs, public article DTOs and pagination. Defines `ProviderArticle`, `ArticleDto`, `Pagination`. Referenced by `gnews.provider.ts`, `news.provider.ts`. |

### `touchline-news-api/src/utils`

| File | Responsibility and connections |
|---|---|
| `deduplicate.ts` | Normalises article fields and evaluates duplicate candidates for ingestion. Defines `normalizeTitle`, `normalizeUrl`. Referenced by `seed.ts`, `news-sync.service.ts`. |
| `logger.ts` | Provides the news service’s logging interface. Defines `logger`. Uses `env.ts`. |

### `touchline-news-api/tests`

| File | Responsibility and connections |
|---|---|
| `app.test.ts` | Regression coverage for app. Uses `app.ts`. |
| `categorization.test.ts` | Regression coverage for categorization. Uses `categorization.service.ts`. |
| `deduplicate.test.ts` | Regression coverage for deduplicate. Uses `deduplicate.ts`. |
| `setup.ts` | Supplies the news test suite with its test environment. |
| `team-detection.test.ts` | Regression coverage for team-detection. Uses `team-detection.service.ts`. |
