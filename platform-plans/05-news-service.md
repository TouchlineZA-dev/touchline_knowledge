# Touchline news service plan

**Program** — `/Users/phathusigama/Documents/touchline_main_fresh/touchline-news-api`, local `main` at `790fc10`.  
**Recommendation** — Keep this Node/Prisma service as the private news provider because it already has articles, sources, team tags, categories, filtering and pagination that Expo uses. Publish news through the C# API so every client uses the same product URL. This adds an internal service to the supplied diagram and preserves news features already present in the apps.

## Reconcile the three existing paths

| Current path | What the code does | Cutover action |
| --- | --- | --- |
| Expo → Node `/api/news` and `/api/categories` | Reads articles and categories from Prisma-backed storage. The Node service can schedule GNews ingestion. | Keep these internal contracts while exposing versioned C# routes to Expo and web. |
| Web → Netlify `news.js` → `sync.news_cache` | Reads `sa_football_news` as refreshed by the sync repo’s scheduled RSS job. | Preserve the South African RSS sources and article URLs in the chosen ingestion pipeline before retiring that job. |
| FastAPI `/api/v1/news/*` | On a cache miss it calls GNews, can merge the Node response and writes per-query cache keys. Its writer references `gnews_cache_ttl_minutes`, which the current settings class does not define. | Identify any active consumers. Fix cache expiry if this path must run during transition, then move consumers and disable overlapping GNews calls. |

## Implement the shared feed

1. **Give news its own storage boundary.** Identify the database currently used by Prisma and how it was initialized. If using the same Supabase Postgres instance, isolate news tables and permissions in a dedicated schema with one migration owner. Keep Flyway ownership of `football`, `sync` and `app`. Rehearse initialization and data transfer before changing a connection string.
2. **Consolidate ingestion.** Bring across the required RSS sources and retain GNews, source metadata, published times, team tags and category slugs. Deduplicate articles by canonical publisher URL. Preserve history and links where in use. Bootstrap categories and a first ingestion run before directing traffic. Configure exactly one scheduled news ingester, a provider-quota-aware interval, retries and last-success monitoring. Retain the last good articles when a provider fails.
3. **Serve news through C#.** Add a private service URL to the API configuration. Add public `GET /api/v1/news` and `GET /api/v1/news/categories` with documented article and pagination fields. Keep `GET /api/v1/news/headlines?max=N` as a compatibility adapter for existing web builds. Align the category filter with the identifiers returned by the categories route. Apply bounded timeouts and expose feed freshness. Separate an empty category from provider or storage failure.
4. **Retire overlapping paths after parity.** Compare selected categories, story counts, source coverage, URLs and timestamps across web and Expo. Switch the clients independently. Observe traffic from older builds before removing legacy read adapters. Transfer schedule ownership once, then remove retired credentials and jobs from their hosting projects.

## Acceptance

The same known article appears through C# on both clients, opens the same publisher URL and appears once after a repeated ingestion. A category filter returns the expected articles, paging has stable ordering, and one failed source leaves the previous feed readable. Disabling the legacy RSS job leaves the next scheduled refresh successful. A news outage leaves football and fantasy available.

**Release sequence across programs** — Begin with [database and worker readiness](01-sync-worker.md), then [the C# identity and fantasy contract](02-platform-api.md). Prove one team across [web](03-web-and-capacitor.md) and [Expo](04-expo-mobile.md). Public football parity and news consolidation can proceed alongside that work. Cut over feature reads after parity, and route fantasy writes to one owner throughout.
