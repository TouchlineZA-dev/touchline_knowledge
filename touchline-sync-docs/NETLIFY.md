# Touchline sync on Netlify (Scheduled Functions)

A serverless port of the sync worker. Two **Netlify Scheduled Functions** pull
from Sportmonks and upsert into the same Supabase database — no always-on host,
runs on Netlify's free cron.

## Who does what

- **Deployer (one-time):** connect this repo to Netlify and set the environment
  variables (below). Do this once. After that it's hands-off.
- **Everyone else:** just `git push` to `main`. Netlify **auto-deploys every
  push** — no dashboard steps for routine changes.

The three env-var **values** (DB URL + Sportmonks key) are secrets — get them
from the repo owner via a password manager, not from the repo. This file lists
*which* variables to set, never their values.

| Function | Schedule | Does |
|---|---|---|
| `sync-fixtures` | hourly | Windowed fixture sync (yesterday → +14d) for the current PSL season, with events/stats/lineups/players |
| `sync-daily` | 00:00 UTC (02:00 SAST) | Refresh competitions/seasons/teams + full-season fixture reconciliation |
| `sync-news` | every 3 hours | Pull SA football RSS feeds (SoccerLaduma, iDiski Times, KickOff, Google News PSL), store latest 50 in `sync.news_cache` (key `sa_football_news`). No API key needed. |

It writes the exact same schema (`football` tables) as the Python worker, using
the same idempotent `ON CONFLICT` upserts and the same Sportmonks quirks
(windowed endpoint, retry/backoff, player dedup, `sportmonks_*` id join contract).

## Deploy

1. **Push this repo to GitHub** (already connected).
2. Netlify → **Add new site → Import from Git** → pick `touchline-sync`.
   Netlify auto-detects `netlify.toml`; no build command needed.
3. **Site settings → Environment variables** — add:

   | Key | Value |
   |---|---|
   | `DATABASE_URL` | Supabase `sync_worker_user` URL — **use the Transaction pooler (port 6543)** for serverless |
   | `SPORTMONKS_API_KEY` | production key |
   | `SPORTMONKS_BASE_URL` | `https://api.sportmonks.com/v3/football` |

4. **Deploy.** The scheduled functions register automatically from their
   `config.schedule` export. First runs happen at the next hour / next 00:00 UTC.

## Verify

- **Netlify dashboard → Functions** — you'll see `sync-fixtures` and `sync-daily`
  listed as *Scheduled*.
- Trigger a test run: **Functions → sync-fixtures → Run** (or wait for the hour).
  A success returns JSON like:
  ```json
  { "ok": true, "mode": "windowed", "season": 28123,
    "window": ["2026-08-07","2026-08-22"],
    "counts": { "fixtures": 4, "players": 0, "events": 0, "stats": 8, "lineups": 0 } }
  ```
- Confirm in Supabase: `SELECT max(updated_at) FROM football.fixtures;`

## Local checks

```bash
npm install
npm run typecheck          # tsc --noEmit
netlify dev                # optional: run functions locally (needs Netlify CLI + .env)
```

## Limits & caveats (read before relying on it)

- **Migrations are NOT run here.** Flyway (`./migrate.sh`, from the Python side)
  still owns the schema. Run migrations against Supabase before first use.
- **15-minute function ceiling.** `sync-fixtures` (windowed) is ~1–2 min — safe.
  `sync-daily`'s full-season reconciliation scans the whole corpus and will creep
  toward the limit as seasons accumulate; split or narrow it if it times out.
- **PSL-only**, like the Python worker — the season resolver targets league 806.
  Covering MTN8/Nedbank/CAF means generalising `getCurrentPslSeasonId` +
  iterating each competition's current season.
- **No `/health`, `/news`, or manual-trigger HTTP API** — those lived in the
  Python app. Netlify functions are the trigger.
- **Failure visibility:** check Netlify's function logs. There's no webhook alert
  like the Python worker's `ALERT_WEBHOOK_URL` (could be added).

## Relationship to the Python worker

This supersedes `app/` for *running the syncs*. The Python code remains the
reference implementation (and still owns migrations via Flyway). Run **one or the
other**, not both against the same DB (they'd just do redundant idempotent work).
