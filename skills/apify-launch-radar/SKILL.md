---
name: apify-launch-radar
description: Cross-platform launch radar that tracks new product and tool launches and how the developer community is reacting, by chaining Apify Actors across Product Hunt, Hacker News, and Reddit. Pulls Product Hunt leaderboards or topic search, matches Show HN / HN search discussion, and Reddit threads, then merges everything into one ranked digest with cross-platform traction (upvotes, points, comments), a sentiment read, and source links. Use when the user asks to monitor product launches, see what launched on Product Hunt this week, watch Hacker News or Show HN for a topic, check what people say on Reddit about a product or competitor, run launch or competitor-launch monitoring, find trending dev tools or AI tools, build a daily/weekly launch digest, gauge community reception of a launch, or track buzz and sentiment for a brand across PH + HN + Reddit. Requires the Apify CLI or the Apify MCP server.
author: Hardy Lebsack
author_url: https://github.com/harryroger798
---

# Launch Radar

Turn "what's launching and how is it landing?" into one ranked, cross-platform digest. The skill chains three Apify Actors — Product Hunt (discovery), Hacker News (developer reception), and Reddit (community discussion + sentiment) — and merges them per product so the agent reports **traction** (upvotes / points / comments), a **sentiment read**, and **source links** instead of three disconnected scrapes.

## Prerequisites
(No need to check this upfront.)

The skill supports two execution paths. Pick the one that matches your environment — the run steps show commands for both.

**MCP path (default in Claude sessions, recommended).** If the Apify MCP server is connected, no setup is needed — auth runs through the user's Apify account. Use the `call-actor` and `get-dataset-items` MCP tools.

**CLI path (scripted / scheduled / non-Claude execution).** Requires the [Apify CLI](https://docs.apify.com/cli) and authentication via `apify login` or an `APIFY_TOKEN` env var ([get a token](https://console.apify.com/settings/integrations)).

## Workflow

Copy this checklist and track progress:

```
Task Progress:
- [ ] Step 1: Collect the radar inputs
- [ ] Step 2: Pick the mode (Discovery vs Tracking)
- [ ] Step 3: Run Product Hunt — get the launch set
- [ ] Step 4: Run Hacker News — match developer reception
- [ ] Step 5: Run Reddit — pull community discussion + sentiment
- [ ] Step 6: Merge per product, score, and render the digest
```

### Step 1: Collect the radar inputs

Ask all of these as one block before any Actor call:

1. **Subject** — a topic/keyword (e.g. `AI dev tools`, `web scraping`) for **Discovery mode**, or one or more specific product/brand names (e.g. `Cursor`, `WarpFix`) for **Tracking mode**. This drives Step 2.
2. **Timeframe** — `daily` (today's launches), `weekly` (past 7 days), or `monthly` (past 30 days). Default `weekly`.
3. **Platforms** — which of Product Hunt / Hacker News / Reddit to include. Default: all three.
4. **Volume** — max items per platform. Default `50` for Product Hunt, `30` for Hacker News, `40` for Reddit. Raise only when the user asks for depth (cost scales with volume — see [references/gotchas.md](references/gotchas.md)).
5. **Output format** — `Markdown digest` (default), `CSV`, or `JSON`.

**Ambiguity rule:** if the subject could be either a topic or a product name, ask **one** follow-up before running. Never burn Actor compute on a guessed mode.

### Step 2: Pick the mode

| Mode | When | Flow |
|---|---|---|
| **Discovery** | Subject is a topic ("what AI tools launched this week?") | PH leaderboard/topic search **finds** the products → HN + Reddit measure reception of each |
| **Tracking** | Subject is a known product/brand ("how did Cursor's launch land?") | Skip PH discovery; query HN + Reddit (and PH search) **for that name** directly |

### Step 3: Run Product Hunt — get the launch set

Discovery mode: scrape the leaderboard for the timeframe. Tracking mode: pass `query` with the product/brand name instead.

**MCP path:** call `call-actor` with `actor: "nexgendata/product-hunt-scraper"` and the input below; capture `datasetId`.

**CLI path:**

```bash
apify actors call "nexgendata/product-hunt-scraper" \
  -i '{"timeframe":"weekly","maxProducts":50,"outputMode":"raw"}' \
  --json \
  --user-agent apify-awesome-skills/apify-launch-radar \
  2>/dev/null
```

For Tracking mode, swap the input for `{"query":"<product or brand>","maxProducts":50,"outputMode":"raw"}`. Keep the `name`, `tagline`, `votesCount`, `url`, `topics`, and maker fields from each item — `name` is the join key for Steps 4–6.

### Step 4: Run Hacker News — match developer reception

For each product name (Discovery) or the subject (Tracking), search HN and also pull recent **Show HN** posts (where launches are announced).

**CLI path — search for a product's reception:**

```bash
apify actors call "harvestlab/hacker-news-scraper" \
  -i '{"mode":"search","searchQuery":"<product name>","maxItems":30,"minPoints":1}' \
  --json \
  --user-agent apify-awesome-skills/apify-launch-radar \
  2>/dev/null
```

**CLI path — sweep recent Show HN launches (Discovery):**

```bash
apify actors call "harvestlab/hacker-news-scraper" \
  -i '{"mode":"show","maxItems":30}' \
  --json \
  --user-agent apify-awesome-skills/apify-launch-radar \
  2>/dev/null
```

Keep `title`, `points`, `num_comments`, `url`, and the HN discussion link per story. Set `includeAiAnalysis` only if the user wants an LLM theme digest — it adds cost (see gotchas).

### Step 5: Run Reddit — community discussion + sentiment

Search Reddit for each product/brand to capture organic discussion and sentiment cues (comment volume + score + tone of titles/snippets).

**CLI path:**

```bash
apify actors call "trudax/reddit-scraper-lite" \
  -i '{"searches":["<product name>"],"maxItems":40,"maxPostCount":40,"sort":"relevance"}' \
  --json \
  --user-agent apify-awesome-skills/apify-launch-radar \
  2>/dev/null
```

To scope to a specific community (e.g. r/SaaS, r/programming), pass `searchCommunityName` instead of relying on global search. Keep `title`, `communityName`, `upVotes`, `numberOfComments`, `url`, and `body`/snippet per post.

### Step 6: Merge per product, score, and render

Pull each dataset (MCP: `get-dataset-items` with the `datasetId`; CLI: the run already printed items). Then:

1. **Join** HN and Reddit results to each Product Hunt product by case-insensitive name match (fall back to fuzzy contains for multi-word names). In Tracking mode there is a single subject row.
2. **Traction score** per product — a transparent, additive signal (don't overfit):
   `score = PH_votes + (HN_points × 2) + HN_comments + Reddit_upvotes + (Reddit_comments × 2)`
   HN points and Reddit comments are weighted up because they reflect engaged developer attention. State the formula in the deliverable so it's auditable.
3. **Sentiment read** — classify each product `positive / mixed / negative / no-signal` from the tone of HN + Reddit titles and top comments. This is a qualitative read, not a model score; quote 1–2 representative lines as evidence. Never invent sentiment for a product with no HN/Reddit hits — label it `no-signal`.
4. **Render** the ranked digest (default Markdown). One row per product, sorted by traction score.

Digest row schema:

| Column | Source |
|---|---|
| Rank | computed |
| Product | PH `name` (or subject in Tracking mode) |
| Tagline | PH `tagline` |
| PH votes | PH `votesCount` |
| HN points / comments | HN aggregated |
| Reddit upvotes / comments | Reddit aggregated |
| Traction score | computed (formula above) |
| Sentiment | positive / mixed / negative / no-signal |
| Evidence | 1–2 quoted lines + source links (PH, HN, Reddit) |

## Actor routing

| User need | Actor ID | Tier | Best for |
|---|---|---|---|
| Discover launches / scrape a PH leaderboard or topic | `nexgendata/product-hunt-scraper` | community | Daily/weekly/monthly leaderboards, topic search, maker data |
| Developer reception, Show HN, HN search | `harvestlab/hacker-news-scraper` | community | HN top/search/ask/show/jobs; points, comments; optional AI digest |
| Community discussion + sentiment | `trudax/reddit-scraper-lite` | community | Reddit posts/comments by search term or subreddit, pay-per-result |

`Tier` = `apify` (Apify-maintained, prefer) or `community` (third-party). Full routing notes and alternates are in [references/actor-index.md](references/actor-index.md).

## Calling Actors — choose your interface

### Option A: Apify CLI (recommended for portability)

Three flags on every call:

```bash
apify actors call "ACTOR_ID" -i 'JSON_INPUT' \
  --json \
  --user-agent apify-awesome-skills/apify-launch-radar \
  2>/dev/null
```

| Flag | Why |
|------|-----|
| `--json` | Stable machine-readable output |
| `--user-agent` | Apify telemetry attribution |
| `2>/dev/null` | Suppress progress messages that break JSON |

Other useful commands:

```bash
# Search the store for an alternate Actor
apify actors search "product hunt" --json --limit 10 2>/dev/null

# Fetch an Actor's input schema before building input
apify actors info "nexgendata/product-hunt-scraper" --input --json \
  --user-agent apify-awesome-skills/apify-launch-radar 2>/dev/null

# Fetch results from a dataset
apify datasets get-items DATASET_ID --format json \
  --user-agent apify-awesome-skills/apify-launch-radar 2>/dev/null
```

### Option B: Apify MCP connector

Hosted MCP server at <https://mcp.apify.com>. Documented at <https://docs.apify.com/platform/integrations/mcp>.

### Option C: MCP client of your choice (e.g. `mcpc`)

Standalone CLI client. See <https://github.com/apify/mcpc>.

## Worked example

See [examples/example-topic-radar.md](examples/example-topic-radar.md) for a full Discovery-mode run ("AI dev tools, this week").

## Quality rules (always enforce)

- **No fabrication.** Only report products/threads the Actors actually returned. A product with no HN/Reddit hits is `no-signal`, never invented sentiment.
- **Auditable scoring.** Always print the traction-score formula in the deliverable so the ranking can be checked.
- **Evidence over adjectives.** Back every sentiment label with 1–2 quoted lines + links, not bare claims.
- **Confirm before expensive runs.** If requested volume pushes the estimate over the gotchas thresholds, warn first.
- **Ambiguity confirm.** If the subject is ambiguous (topic vs product), ask one follow-up before running.

## Troubleshooting

- **Product Hunt returns nothing for a topic** → switch from leaderboard mode to `query` search, or widen the timeframe (`weekly` → `monthly`).
- **HN/Reddit name match is noisy** (common short names like "Arc", "Notion") → add a qualifier to the search (`"<product> app"`, `"<product> launch"`) or scope Reddit with `searchCommunityName`.
- **Reddit run is slow / large** → lower `maxItems` and `maxPostCount`; Reddit is pay-per-result.
- For cost guardrails and per-Actor quirks, see [references/gotchas.md](references/gotchas.md).
