# Handoff fixes for the API's Sprint 9

Prepared 24 Nov 2026 for Leruo, answering his 13 points. Branch `sprint/6-api-handoff-fixes`, on top of `main@b1a60f2`. Builds on Sprints 3–5 ([live](../sprint3/live-freshness.md), [late changes](../sprint4/late-changes.md), [operations](../sprint5/operations.md)).

## Migration order (read before anyone migrates)

| Version | What | Note |
|---|---|---|
| V039–V041 | drift, live freshness, corrections and completeness | already on main |
| V043–V048 | API team's migrations | already on main |
| **V049** | Sprint 5 operations | **was V042.** Renumbered so it runs after V043–V048 without `-outOfOrder`. It touches none of their objects |
| **V050** | fixes below | additive; replaces functions and adds one view |
| **V001** | passwords removed | **checksum changes**, see §8 |

Verified on Postgres 17:

**Clean install, V001→V050:**
- every migration succeeds
- 152 Python tests and 23 TypeScript tests pass, and `tsc` is clean
- neither login role has a password until one is set

**Upgrade shaped like production** (old V001…V038, then this branch):
1. A plain `migrate` refuses, with a checksum mismatch on 001.
2. `./migrate.sh repair` once, then `./migrate.sh migrate`, applies V039→V050.
3. `validate` passes.
4. The existing role passwords are untouched.

**Production today** (read-only check): nothing from V039 onward is applied:
- no `withdrawn_at` columns
- no V040–V048 tables
- V039's revokes haven't run: the worker can still read `app.users` and write `leaderboards`, `player_prices` and `points_results`

## The 13 points

**1. V042 → V049.** Done; see the table above.

**2. Own goals.** Leruo is right: `GET /v3/core/types` gives **14 Goal, 15 Own Goal, 16 Penalty, 17 Missed Penalty.**
- The Sprint 4 check of "own goals on the benefiting team (29 of 29)" had actually counted penalties.
- Rechecked on type 15: **16 of 16** production own goals carry the benefiting `team_id`, with the scorer in the other team's lineup, and the scores reconcile. So storage was always right; only the label was wrong.
- The wrong label caused one real bug: V041 required a scorer only for types 14 and 15, so **an unattributed penalty could pass as complete.**
- V050 (rule version 2) requires a scorer for every goal type, and ignores goals the provider marks `rescinded`.
- `ops_status.py` labels and the docs are fixed.

**3. Data feed.**
- **Events and lineups didn't stop on 1 Nov.** No PSL match has been played since 31 Oct: the seven fixtures on 4 Nov are postponed (state 10), and the stored schedule has nothing else before 23 Nov. The stored schedule itself stopped updating on 18 Nov, when the key lost PSL access.
- **2025/26 backfill:** the season has 240 fixtures and **0** with lineups (2024/25 has 221 of 240). Once the key has PSL access, run `python scripts/reconcile.py --season 26173`. That is one full-season job as `operator`: about 4 league-filtered chunks of 1–2 pages, so a handful of calls. It's tested against a past season.

**4. One live writer.** Recommendation: **Netlify**. It is the only writer that has ever run against production. The Python worker stays deployable with `SCHEDULER_ENABLED=false`, and V049's lease enforces one scheduler anyway. Cut-over is recorded in `sync.sync_jobs.runner` and `sync.scheduler_lease` from the first tick after V049. **Phathu to confirm.**

**5. `gameweek_sync_status` has 0 rows.** Only the Python worker writes it, and production has always been written by Netlify. **Yes, it is replaced.** V050 adds **`sync.gameweek_completeness`**, readable by `app_api_user`:
- one row per gameweek, rolled up from `sync.fixture_completeness` over the fixtures **assigned** to it
- counts per status
- **`ready`**: at least one fixture is complete and every other one is complete or excluded
- **`needs_admin`**: a fixture is permanently unavailable

Score from `ready`, and rescore from `sync.late_corrections`.

**6. Postponed fixtures.** These were already excluded in V041 and are unchanged:
- states 10, 12, 13 and 20 are `excluded`, never `incomplete`
- they count as done in `gameweek_completeness.ready`
- abandoned (15) is `unavailable`, because it needs a ruling

**7. `app.fixture_gameweek_assignments`.**
- **Worker access:** the worker gets `SELECT` plus an RLS `SELECT` policy. V043's only policy was for `app_api_user`, so a grant alone would have returned no rows. The worker still can't insert (tested).
- **`sync.fixture_gameweek(fixture_id)`:** the newest assignment row wins, even when its gameweek is NULL (unassigned). The SAST date window applies only to a fixture that has never been assigned.
- Completeness uses it.
- **Reschedule history:**
  - `old_gameweek_id` is the **assigned** gameweek.
  - `new_gameweek_id` is where the date rule would put the new kickoff, or NULL once the fixture is postponed, cancelled, TBA, abandoned or deleted.
  - The worker never writes assignments; moving a fixture stays the API's decision.
- **Behaviour change:** a postponement row now has `new_gameweek_id = NULL`, where V041 had the old gameweek.

**8. Passwords.**
- **V001 and `db/supabase_schema.sql` no longer contain passwords.** The roles are created without one, and an admin sets it with `scripts/rotate_role_passwords.py`.
- The old values are in git history for good, so **treat them as public.**
- **The production worker password is not the V001 one, and it appears nowhere in git history.**
- **Not checked:** whether production's `app_api_user` still uses its V001 password. Rotate both roles anyway, with `scripts/rotate_role_passwords.py` as admin (runbook §11), then update every `DATABASE_URL` and the API's connection string.
- **Before the next migrate on any existing database, run `./migrate.sh repair` once,** because V001's checksum changed.

**9. Dependencies and secrets.**
- **Python:** the 3 vulnerable pins are bumped: `anyio` 4.14.2, `certifi` 2026.10.18, `urllib3` 2.8.0. pip-audit now finds none.
- **Dependabot's 14:** consistent with the 7 advisories counted once for `requirements.txt` and again for `requirements-dev.txt`, which includes it. That still needs confirming on GitHub.
- **npm:** `npm audit` reports 0.
- **Full-history secret scan** (gitleaks 8.21, every ref, 64 commits): one hit, a deliberately fake key in `tests/test_security.py`, now marked `gitleaks:allow`.
- Every secret in the on-call `.env` was searched across all history: **none found**. The only real credentials ever committed are the two V001 passwords.

**10. `BACKEND_INTEGRATION.md`.**
- **Budget:** default 100.0 (V035), with R100m vs R300m still pending sign-off.
- **States:** all 25, from `GET /v3/football/states`, which is the endpoint that works; `/v3/core/states` returns nothing.
- **Scoring types:** 14 and 16 are goals, 15 an own goal on the benefiting team, and rescinded or withdrawn events are skipped.
- **Short codes:** proposed fallbacks for the 5 clubs Sportmonks gives none: Durban City `DUR`, Kruger United `KRU`, Milford `MIL`, Polokwane City `POL`, Siwelele `SIW`. Apply them in the API, keyed on `sportmonks_team_id`, because the worker overwrites provider fields.

**11. Provider timestamps.** This is already done as far as Sportmonks allows.
- Re-probed today: fixtures and events carry **no** update timestamp.
- The only provider times are `periods[].started/ended`, which V040 stores as `sync.fixture_revisions.provider_at`.
- `sync.fixture_revision_latency.provider_to_commit` measures provider delay on phase changes.
- For goals, only observation-to-commit can be measured.

**12. Responders.** `unresolved_ids` and `incomplete_fixtures` are already seeded as Leruo (api-owner). **Phathu to confirm** the sync-oncall and provider-account roles. A DB admin then edits `sync.ops_responders`.

**13. `phathu-dev` in touchline-api.** Noted: **don't merge it.** It has R300m and Triple Captain ×9, while main has R100m and ×4.

## Still with Phathu

- **The Sportmonks PSL subscription.** It blocks the backfill, real latency figures and every live test.
- The writer choice (Netlify recommended).
- Confirming the responders.
- Rotating the role passwords.
- The FINAL-SUMMARY sign-offs.
