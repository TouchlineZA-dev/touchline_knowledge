# Touchline Expo mobile plan

**Program** — `/Users/phathusigama/Documents/touchline_main_fresh/mobile-app`, local `main` at `790fc10`.  
**Outcome** — Expo acts as a second client of the [platform API](02-platform-api.md). A fan can move between a phone and the web without creating a new account or fantasy team.

## Preserve what already works in the code

`src/lib/auth/AuthContext.tsx` signs in with Supabase and listens for session changes. `src/lib/api/client.ts` already sends bearer tokens on protected calls and implements team creation, snapshot reads, squad saves and several league operations. Live and table screens load football data. These are useful starting points for the first staging journey. The league tab still shows a coming-soon banner, the squad formation selector changes screen state, and the player market browses players. A client method alone does not complete a screen.

## Changes in delivery order

1. **Make builds target the right environment.** In `src/lib/config/env.ts`, require the public Supabase configuration and the platform API URL for release builds. Remove the baked-in project URL and placeholder key fallback. Keep emulator-only defaults in development. On a physical phone use reachable staging HTTPS or a configured development host. Retire `EXPO_PUBLIC_NEWS_API_BASE_URL` once the central news route is available. Fix the observed type-check failure at `components/ExternalLink.tsx:11`, where a generic string is passed to a typed Expo Router link, and rerun the mobile type check.
2. **Finish identity and onboarding.** After restoring a valid session, resolve the profile with `/me`. The mobile sign-up flow supplies username metadata while the current API provisions a generated username. Use the agreed profile edit/read contract for the visible name and club. Handle email confirmation, expired sessions, account conflicts and sign-out. If social sign-in ships, validate the registered app scheme and callbacks on a device against the same Supabase project as the web.
3. **Finish squad editing.** Connect the market to a draft of 15 players. Collect a formation, starters, ordered bench, captain, vice-captain and the intended gameweek number. Submit the first pick through the existing `saveSquad` method. Wire subsequent changes to `PUT /fantasy/team/lineup` and `POST /fantasy/team/transfers`. Show the API’s remaining budget, validation details, deadlines, chip usage and transfer cost. Refresh the snapshot after each accepted write.
4. **Handle cross-device edits.** The request helper currently returns only decoded JSON and drops response headers. Capture the squad ETag, send it with lineup and transfer edits, and handle 409 with the latest selection. If conditional GETs are enabled, handle 304 before generic HTTP error handling. Keep cached private state scoped to account and season.
5. **Connect the remaining tabs.** Build league list/create/join/leave and standings around the API, adding the missing leave action. Use the shared PSL standings response in place of the table tab’s fixture aggregation. Switch news and category requests to C# while preserving pagination/filter behavior. Use the canonical match detail and data freshness in the live screen. Keep PSL table standings and fantasy league standings distinct in navigation.

## Acceptance on a phone

- Use the same staging account as the web. The profile, team ID, 15 picks and balance agree after a cold start.
- Change captain on the phone and reload on web. Make concurrent edits against the previous squad version. One accepted edit persists and the other client receives a recoverable conflict.
- Join a private league from the phone. Verify the same membership and gameweek points on web. A second account sees only its own team and authorized leagues.
- Exercise no team, no upcoming gameweek, expired session, offline snapshot, delayed scores and unavailable news. Each screen distinguishes those states and offers the appropriate action.

**Dependency** — Ship the authenticated team journey once migrations and the API are ready. Release full mobile news, standings and fantasy actions as their public contracts pass staging. Coordinate existing Capacitor installs through the [web and Capacitor plan](03-web-and-capacitor.md).
