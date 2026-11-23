# Touchline web and Capacitor plan

**Program** — `/Users/phathusigama/Documents/touchline_main_fresh/frontend`, local `main` at `790fc10`.  
**Outcome** — A fan signs in on the web or packaged app, sees the same football data as Expo, and can manage the same fantasy team.

## Fix sign in and routing first

1. Replace the local account registry in `src/services/authService.ts` and wire `src/context/AuthContext.tsx` and the auth modal to Supabase Auth. The current code stores the submitted password unchanged under a field named `passwordHash`, creates a browser-local user ID, and restores a local profile. Stop collecting passwords through that path. On upgrade, clear legacy credentials on each device and ask the fan to authenticate. Transfer eligible non-secret preferences or squad drafts only after verification and user confirmation. Never upload the old passwords or use an email match to establish ownership.
2. Use the environment’s public Supabase URL and publishable/anon key. Restore the session, handle confirmation, refresh and sign-out, and pass its access token to C# for `GET /api/v1/me` and fantasy actions. Present the API’s account-conflict recovery. Keep display name, favourite club and team name consistent with the profile/team contracts in the [API plan](02-platform-api.md).
3. Consolidate the base URL in `src/services/touchlineApi.ts`. Currently the browser defaults to relative `/api/v1`, `vite.config.ts` proxies `/api` to port 8000, and the native branch hardcodes a Netlify host. Use the dedicated C# API for all three environments. For development, proxy to its port 5147. For a deployed site, route the public prefix to the API before the single-page fallback. Use a reachable HTTPS API origin in the Capacitor build.

## Move feature reads without breaking the live screens

| Screen | Work in this program | Acceptance |
| --- | --- | --- |
| Live fixtures and match detail | Map C# `kickoffUtc` into the display model, resolve crests, and handle status and scorer differences. Move match detail when the API supplies events, lineups and stats. Preserve UTC input and SAST display. | The same match has the same score, time and timeline on web and mobile. |
| PSL table and player rankings | Keep the existing functions for reads until the C# routes match their filters, ordering and counts. Compare a populated season and an empty one. Update player cards that expect generated ratings. | Standings and leaders agree for the same season. An absent value displays as unavailable. |
| News | Move `fetchNews` from the Netlify cache reader to the C# news contract or compatibility route. Retain headline images, source links, timestamps and empty/error states. | A known story is visible on both clients and opens the same publisher URL. |
| My Squad and Leagues | `FootballDashboard.tsx` still presents squad management as coming soon. Deliver real screens for team creation, picks, captaincy/bench, transfers, chips, league membership and standings. Save the first selection through `PUT /fantasy/team/squad`. Use the distinct lineup and transfer routes for later edits. | A web-created squad survives sign-out and is editable in Expo. Server validation errors identify the invalid picks. |

Make one typed, authenticated API client shared with Expo where practical. Keep platform-specific session storage and navigation in their apps. Store cached snapshots under user and season. Treat the API as authoritative for prices, balance, points and deadlines. Read `ETag`, send `If-Match` for lineup/transfer edits, reload on 409 and preserve the user’s ability to review the change. Clear private caches on sign-out.

## Retire the old paths

Inventory usage of the deployed Netlify redirects, including previously released Android builds that might still call the hardcoded host. Maintain compatible routes there until those versions have migrated. Check whether the legacy fantasy function ever stored real teams under `owner_auth_user_id` and `fantasy_team_picks`. If it did, record ownership, map to the canonical profile/team/gameweek IDs, rehearse the import and compare team/pick counts. Route all team writes to one backend from the cutover onward. Disable legacy write handlers even if old clients still reach their URLs.

After read parity, remove the per-feature Netlify handlers, server database credentials from the frontend hosting project, duplicate schema deployment paths and the embedded `Touchline API/` from active build instructions. Keep a compatible read route available for rollback. Verify both a browser and an installed Capacitor build against staging, including token expiry, unavailable API, session restoration and a cross-device edit.
