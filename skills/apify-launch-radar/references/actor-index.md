# Actor index — apify-launch-radar

Routing table the agent reads after `SKILL.md` to pick the right Actor for a
specific user intent. All Actors are publicly available on the
[Apify Store](https://apify.com/store).

| Platform | User intent | Actor ID | Tier | Notes |
|----------|-------------|----------|------|-------|
| Product Hunt | Discover today's / this week's / this month's launches, or search a topic | `nexgendata/product-hunt-scraper` | community | Leaderboard mode via `timeframe` (`daily`/`weekly`/`monthly`) + optional `date`; topic mode via `query`; `maxProducts` 1–1000 (default 30); `outputMode` `raw`/`tracker`. Returns name, tagline, votes, url, topics, makers. |
| Hacker News | Developer reception, Show HN launches, HN full-text search | `harvestlab/hacker-news-scraper` | community | `mode` one of `top`/`search`/`ask`/`show`/`jobs`/`user`; `searchQuery` (search mode); `maxItems` 1–500 (default 30); `minPoints` filter; `includeAiAnalysis` adds ~$0.05/run. ~$0.001/item. Uses official Algolia + Firebase HN APIs (no key). |
| Reddit | Community discussion + sentiment signal | `trudax/reddit-scraper-lite` | community | `searches` (keywords) **or** `startUrls` (subreddit/post URLs); `searchCommunityName` to scope to one community; `maxItems`, `maxPostCount`, `maxComments`. Pay-per-result. Don't combine `searches` with `startUrls`. |

## Alternates (if a primary Actor is unavailable or rate-limited)

| Platform | Alternate Actor ID | Notes |
|----------|--------------------|-------|
| Product Hunt | `diverse_venture/producthunt-scraper` | `today`/`date` scrape modes + leaderboards |
| Product Hunt | `kawsar/product-hunt-scraper` | Simpler search-term scraper |
| Hacker News | `junipr/hacker-news-scraper` | Firebase + Algolia, full comment threading |

## How to extend

1. Search for candidates: `apify actors search "KEYWORDS" --json --limit 20 2>/dev/null`
2. Fetch input schema: `apify actors info "ACTOR_ID" --input --json 2>/dev/null`
3. Add a row above with the user intent that should trigger it.

Before swapping in any alternate, re-check its current input schema with
`apify actors info` — community Actor inputs change over time.
