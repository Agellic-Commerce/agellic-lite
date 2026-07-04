# Tool Reference

This document is the practitioner reference for the 8 MCP tools exposed by
Agellic Lite v1.5.0, the free, bring-your-own-Keepa-key edition of the
Agellic MCP server. Each section covers what the tool does, what it costs
in Keepa tokens, what inputs it accepts, what it returns, and the operating
rules worth knowing before you turn it loose on a candidate set. All Keepa
token costs are concrete numbers measured against current runtime behavior,
no hedging.

Agellic Lite pulls every figure from Keepa (`keepa.com`) and your Keepa
subscription supplies the token bucket. If you don't have a Keepa account
yet, grab one at [keepa.com/#!api](https://keepa.com/#!api) before you
start. Base Keepa gives you **1 token per minute** (a **60-token bucket**,
since capacity = tpm × 60). At that rate the background queue is the normal
path for anything but the smallest calls, not an error state: a call that
can't be funded up front is queued and drains automatically as Keepa tokens
refill, and the queue is durable across restarts (quitting pauses it,
relaunching resumes it). A free `/token` boot probe auto-detects your key's
real rate on first use, so a higher Keepa tier unlocks a proportionally
larger bucket without hand-tuning. Several sections below note where the
1-TPM bucket is the binding constraint for sync vs background-job behavior.

## Quick scan

| Tool | One-line summary | Token cost |
|------|------------------|------------|
| `execute_keepa_finder` | Discover ASINs by category / brand / price / rank / competition filters; optional market-insights stats. | 10 base + 1 per 100 ASINs returned (stats: +30 base + 1/M total matches) |
| `get_product_details` | Deep per-product analysis: offers, Buy Box rotation, stock depth, calibrated demand, insights. Also resolves UPC/EAN/ISBN → ASIN. | ~8 per uncached ASIN (code lookup: 1 per candidate up to `codeLimit`) |
| `resolve_codes` | Bulk-resolve supplier UPC/EAN/GTIN/ISBN codes (up to 500 rows) to candidate ASINs at the identity tier. | ~1 per returned candidate (≈1/code typical) |
| `get_product_chart` | Fetch a price/BSR history PNG chart for one ASIN on one marketplace. | 1 flat per call (Keepa 90-min server cache) |
| `check_token_balance` | Show current Keepa token balance, refill rate, and cache-aware per-tool cost projections. | 0 (local) |
| `check_job_status` | Check, list, or cancel background jobs created when a call hit a Keepa rate-limit wait; pending status shows queue position + token-wait ETA. | 0 (local) |
| `get_finder_result` | Page through ASINs from a stored finder result set by id. | 0 (local) |
| `get_codes_result` | Page through a stored code-resolution result set (the per-row candidate table). | 0 (local) |

The 4 free tools (`check_token_balance`, `check_job_status`,
`get_finder_result`, `get_codes_result`) read local state only and never
charge Keepa tokens. Cached re-reads of the paid tools are also free for
24 hours.

Domain values referenced throughout: `1`=US, `2`=UK, `3`=DE, `4`=FR,
`5`=JP, `6`=CA, `8`=IT, `9`=ES, `10`=IN, `11`=MX, `12`=BR.

---

## `execute_keepa_finder`

Discovery search across Amazon's catalog. Returns ASINs matching
category / brand / price / rank / competition filters, plus optional
market-insights stats when you ask for them.

**You speak in natural language.** Describe what you're after ("kitchen
items under $40, rating ≥ 4.2, 2–5 sellers, BSR < 10K") and the
assistant picks the Keepa filter names, scales (cents not dollars, mm
not inches), and time windows for you. The filter reference below is
what the LLM consults to pick names and units, not a syntax you type
yourself. The "Natural-language to filter mappings" subsection at the
end of the section shows sample translations.

### What it's good for

- The user has no ASINs yet and wants to find products matching criteria.
- The user wants to **size a market**: fetch `includeStats=true` on a
  follow-up call after a filter-only recon to get avg Buy Box price,
  seller counts, Amazon share, brand fragmentation, FBA share, avg
  rating, and avg review count across the matched set.

Not the right tool when the user already has ASINs: go straight to
`get_product_details`.

### Token cost

- **Base: 10 tokens** + **1 token per 100 ASINs returned**.
- **Stats (`includeStats=true`):** +30 base + 1 per million `totalResults`.
- `perPage` is a **ceiling on cost, not a fixed charge**. Keepa bills
  `10 + ⌈min(perPage, totalResults)/100⌉`. So `perPage=10000` on a
  4,231-match query costs **53 tokens**, not 110. Even the worst case
  (`perPage=10000` returning a full 10,000 ASINs) is 110 tokens.
- A broad **single-filter** search can match hundreds of millions of
  products and cost 300+ tokens. Always combine ≥ 2 filters.

### Inputs

All filters are ANDed; array filters are ORed (max 50 entries each). At
least one real filter is required besides `domain` / `page` / `perPage`
/ `sort` / `stats`.

Common numeric range filters (use `_gte` / `_lte` suffixes):

- `current_BUY_BOX_SHIPPING`: Buy Box price in cents ($40 → 4000). The
  default when the user says "price".
- `current_SALES`: Best Sellers Rank (lower = better).
- `current_RATING`: 0–50 scale (4.5 stars → 45).
- `current_COUNT_NEW`: Listed new-seller count. Default when the user
  says "sellers".
- `delta30_BUY_BOX_SHIPPING`, `delta90_BUY_BOX_SHIPPING`: Price delta
  vs avg (positive = price dropped).
- `deltaPercent30_SALES`, `deltaPercent90_SALES`: BSR delta percent
  (positive = rank improved).
- `packageWeight` (grams; 1 lb = 454g), `packageHeight` / `packageLength`
  / `packageWidth` (mm; 1 in = 25.4mm).
- `outOfStockPercentage90`: Percent of time out of stock.
- `buyBoxStatsAmazon`: Percent of time Amazon holds the Buy Box.
- `buyBoxStatsSellerCount30/90/180/365`: Unique Buy Box winners.
- `buyBoxStatsTopSeller`: Top seller's BB win share (low = rotating).
- `buyBoxIsAmazon`, Boolean: Amazon currently holds the Buy Box.

String / array filters:

- `brand`, `title`, `manufacturer`: prefer `brand` unless the user
  explicitly says "manufacturer".
- `rootCategory`, `categories_include`: numeric IDs only. Use only when
  you have a trusted ID from a prior tool result or explicit user input.
- `categoryHint`: text description (e.g. `"kitchen"`, `"Bücher"` on
  DE). Resolved server-side via fuzzy match. Single match → injected as
  `rootCategory` / `categories_include`; multiple → ambiguity error with
  candidates; zero → actionable error. Never pass `categoryHint`
  together with `rootCategory` / `categories_include`.

Price stability and flip filters:

- `buyBoxStandardDeviation30/90/365`: Buy Box volatility in cents.
  `_lte` for stable, `_gte` for volatile.
- `flipability30/90/365` (0–255): dip+rebound score. `_gte` for
  swing-prone, `_lte` to exclude.
- Timeframes: 30d = recent, 90d = typical sourcing window, 365d =
  seasonal patterns.
- High flipability + moderate std dev often reflects stable pricing with
  occasional dips, a useful sourcing signal.

Competition and seller field routing:

- "How many sellers?" → `current_COUNT_NEW` (listed). Not
  `buyBoxStatsSellerCount{30,90,180,365}` (BB winners).
- "Is Amazon on this?" → `buyBoxIsAmazon` or `buyBoxStatsAmazon` (%
  over time). Not `outOfStockPercentage90`.
- "Amazon OOS?" → `outOfStockPercentage90`. Not `buyBoxStatsAmazon`.
- "Competitive Buy Box?" → `buyBoxStatsTopSeller_lte` (low = rotating).
- "Buy Box winners?" → `buyBoxStatsSellerCount{30,90,180,365}` (default
  90).
- Seller-count delta: `delta = avg − current`. `delta30_COUNT_NEW_gte: 3`
  means 3 sellers LEFT (avg was higher than current).

### The full filter surface

The fields above are a sampler. The strict validator accepts over a
thousand named filters, generated from a handful of patterns crossed
with Keepa's price types and time windows:

- `current_<TYPE>_{gte,lte}`: current value
- `avg{7,30,90,180,365}_<TYPE>_{gte,lte}`: windowed averages
- `delta{1,7,30,90,Last}_<TYPE>_{gte,lte}`: absolute delta vs that
  window's avg (or previous value for `deltaLast`)
- `deltaPercent{1,7,30,90}_<TYPE>_{gte,lte}`: percent delta
- `backInStock_<TYPE>`: was OOS in last 60d, now has an offer
- `isLowest_<TYPE>` / `isLowest90_<TYPE>`: current is the all-time / 90d low
- `lastPriceChange_<TYPE>_{gte,lte}`: timestamp filter (Keepa minutes)

`<TYPE>` covers the 24 base Keepa types (AMAZON, NEW, USED,
BUY_BOX_SHIPPING, NEW_FBA, NEW_FBM_SHIPPING, COUNT_NEW, COUNT_USED,
COUNT_REVIEWS, RATING, SALES, LISTPRICE, COLLECTIBLE, REFURBISHED,
WAREHOUSE, LIGHTNING_DEAL, TRADE_IN, …); `avg`, `deltaPercent`,
`isLowest`, and `lastPriceChange` also accept 5 extras
(EBAY_NEW_SHIPPING, EBAY_USED_SHIPPING, PRIME_EXCL, RENT,
BUY_BOX_USED_SHIPPING).

Anything that doesn't fit a top-level pattern goes in `advancedFilters:
{ ... }`, a passthrough map of `Keepa-field → value`. The strict layer
validates either way: bad names come back as actionable errors, not
silent drops, so the working posture is **try the filter and adapt from
the response** rather than refuse because a field isn't named in the
section above.

### What it returns

- **Complete result set** (`asins.length === totalResults`): emits a
  bracketed handle line: `[handle: <id> · <count> products · expires
  24h]`. The id is machine metadata; the assistant talks in counts and
  filters, not raw ids. Downstream tools accept the id directly.
- **Partial slice** (`asins.length < totalResults`): emits
  `[partial: <id> · <fetched> of <total> · <advisory>]`. The slice is
  cached too, but the label says refine, or get explicit user
  acceptance, before any downstream tool call uses it.
- **Rate-limited:** returns a `jobId`. The search is queued as a
  background job; poll with `check_job_status`. The completed status
  returns a handle only when the full result set fit on one page.
- **Zero results:** plain-text message; the model suggests filter
  adjustments.

### perPage

`perPage` defaults to **1000**. Range: 50 to 10,000. For queries with
`totalResults ≤ 1000`, the default produces a complete handle on the
first call. For 1,000–10,000 `totalResults`, re-call with
`perPage=10000` to upgrade the partial to a complete handle. Above
10,000, refine: no `perPage` value reaches a complete result set, and a
partial slice of an over-cap query isn't actionable downstream without
explicit user acceptance.

### Operating discipline

- **Iterative workspace.** Finder is a loop, not a one-shot. Start
  broad, narrow with the user, accept the slice the user actually wants.
  Exit condition is user satisfaction, could be 20 products, could be
  9,000.
- **10,000 is a ceiling, not a target.** Keepa caps a single page at
  10,000 ASINs; queries returning more than that cannot be captured as a
  complete result set without refinement. Above 10K, the right move is
  tighter filters (tighter price band, tighter BSR band, higher OOS
  threshold, a sub-category, etc.), not a top-N slice, unless the user
  explicitly prefers a slice.
- **`includeStats=true` is the primary refinement instrument** when
  intuition isn't enough. Insights summarize the matched set so the next
  filter move is obvious. Never set on the first call: run filters-only
  first to learn `totalResults`, then decide whether stats are worth the
  cost. Above 50M matches the cost gets steep; refine before requesting.

### Data and unit invariants

- Prices: integer smallest currency unit. `$50` → `5000`.
- Weights: grams. "Under 2 lbs" → `packageWeight_lte: 907`.
- Dimensions: millimeters. "Under 6 inches" → `packageHeight_lte: 152`.
- Ratings: 0–50 (stars × 10). 4.5 stars → `45`.
- Percentages: integers 0–100 unless noted.
- Delta sign: positive delta = price DROPPED (price-like) or rank
  IMPROVED (BSR).

### Natural-language to filter mappings

- "under $40" → `current_BUY_BOX_SHIPPING_lte: 4000`
- "$20 to $50" → `current_BUY_BOX_SHIPPING_gte: 2000,
  current_BUY_BOX_SHIPPING_lte: 5000`
- "at least 4.2 stars" → `current_RATING_gte: 42`
- "BSR under 10,000" → `current_SALES_lte: 10000`
- "2 to 5 sellers" → `current_COUNT_NEW_gte: 2, current_COUNT_NEW_lte: 5`
- "under 2 lbs" → `packageWeight_lte: 907`
- "under 6 inches tall" → `packageHeight_lte: 152`
- "price dropped by at least $10 vs 30d avg" →
  `delta30_BUY_BOX_SHIPPING_gte: 1000`
- "rank improved by at least 20% vs 30d avg" →
  `deltaPercent30_SALES_gte: 20`

---

## `get_product_details`

Deep product analysis: offers, Buy Box rotation, stock depth, calibrated
demand range, sales rank history, trend analysis, rank volatility,
review velocity, insights, and economics. Also resolves a single
UPC/EAN/ISBN code to ASINs on one specified marketplace.

### What it's good for

- **Deep dive on a shortlist.** The user has a shortlist (from a finder
  result, an external list, or an explicit set of ASINs) and wants
  per-seller offers, Buy Box rotation, stock depth, or the calibrated
  demand range.
- **Single-product identification by code**: UPC/EAN/ISBN on one
  specified marketplace.

### Token cost

**Enriched fetch (`asins` / `resultSetId` modes):**

- **~8 tokens per uncached ASIN** observed average. The tool reserves 9
  tokens per ASIN as a ceiling (6 offers + 2 stock + 1 graph) and
  refunds the difference; the 1-token graph allowance is refunded
  whenever Keepa does not return graph data, which is most calls.
- **0 tokens for cached ASINs** (24h product cache).
- Partial cache hits: only uncached ASINs cost tokens.

**Code lookup (identification tier):**

- **`codeLimit` tokens per call (default 3).** Keepa charges 1 token per
  returned candidate, capped at `codeLimit`.
- No offers / stock / rating enrichment in code mode.

### Inputs

Mutually exclusive, pass exactly one:

- **`resultSetId`**: id from a prior `execute_keepa_finder` or
  `get_product_details` call. The tool resolves ASINs server-side. Max
  50 ASINs in the stored set (narrow first if larger).
- **`asins`**: up to 50 ASINs with full enrichment. Use when you have
  an explicit list.
- **`codes`**: resolve exactly one UPC/EAN/ISBN/GTIN to matching ASINs
  on the specified marketplace. The schema accepts an array for forward
  compatibility, but >1 is rejected at runtime.

Pass `update=null` unless the user explicitly asks for fresh data.
`update=0` forces a live Keepa crawl (~8 tokens per ASIN) even if the
data was fetched minutes ago, the 24h product cache is correct for
almost every workflow.

Every successful fresh fetch persists a `lookup:<timestamp>` result set,
so a follow-up call can re-read the same ASINs by id at 0 tokens.

### What it returns

Per resolved ASIN:

- **`identity`**: title, brand, category, manufacturer, `productCodes`
  (UPC/EAN/GTIN), Amazon URL, image URL.
- **`pricing`**: current lowest new price, list price, Buy Box price,
  30/90/180d averages, trend direction/strength, volatility score.
- **`sales`**: current and historical sales rank, primary + leaf BSR
  with category names, 30/90/180d drops, Amazon-reported monthly sold
  badge (when available; the model's range estimate lives in `demand`).
- **`demand`**: calibrated monthly-sales range (`rangeLow` /
  `centerLikely` / `rangeHigh`), confidence (`high` / `medium` /
  `low`), and `mode` (`read` / `badge` / `no-read`, the discriminant
  that gates which other fields are present, read it first), plus
  `caveats` (free-text qualifiers like "estimate from cross-marketplace
  baseline").
- **`competition`**: individual seller offers (FBA/FBM, prices, stock
  depth), Buy Box current winner, dominant seller + win %, rotation
  table, historical avg seller count.
- **`reviews`**: rating, review count, velocity (added 30/90/180d),
  trend (accelerating/steady/slowing), historical avg.
- **`supply`**: Amazon OOS 30/90/180d, marketplace OOS 90d.
- **`insights`**: rank volatility, trend signals, Buy Box volatility,
  effective competition (sellers within 5% of BB), IP risk,
  race-to-bottom warning. See
  [`COMPUTED-INSIGHTS.md`](COMPUTED-INSIGHTS.md) for the algorithms
  behind every field in `demand`, `seasonality`, and `insights`, what
  each measures, the constants, and how to read it.
- **`economics`**: referral fee percent, FBA pick & pack fee, return
  rate.
- **`metadata`**: listing age, Subscribe & Save eligibility.
- **`tokensUsed`**: actual tokens consumed by this call.

When rate-limited, returns a `jobId`; poll with `check_job_status`. Once
the job completes, re-call `get_product_details` with the same `asins` /
`resultSetId` and cached data is served at 0 tokens.

### Cost discipline

State the worst-case spend before running on a stored result set
(`uncached_asins × 8` tokens, the cache absorbs anything fetched in the
last 24h). Per-product output is **~5–10 KB** depending on offer count
(up to 20 offers per product) and insight verbosity. 10 ASINs ≈ ~75 KB
/ ~20K tokens of context; 50 ASINs ≈ ~375 KB / ~100K tokens. There's
no fixed "batch size target": state the spend, then let the user decide
the batch size. On base Keepa (1 TPM, 60-token bucket) roughly six
uncached ASINs exhaust the bucket, so larger batches queue as a
background job and drain as tokens refill.

### Limitations

- **50 ASINs per call, hard cap.** For larger candidate sets, narrow
  with `execute_keepa_finder` filters (or take a BSR-sorted slice via
  `get_finder_result`) and deep-dive the shortlist.
- **`codes` mode is single-marketplace only.** It resolves a code to
  ASINs on one specified marketplace and does not compare across
  marketplaces.
- **ASINs are NOT globally unique.** The same ASIN can mean different
  products on different marketplaces. Code-mode is for identification on
  one specified marketplace, not for cross-marketplace lookup.

### Data and unit invariants

- **Prices** are integers in the smallest currency unit (e.g. `1999` =
  $19.99 on US domain). Divide by 100 to render.
- **Ratings** use a 0–50 scale (stars × 10). `45` = 4.5 stars. Divide by
  10.
- **Keepa time values** are minutes since 2011-01-01 00:00 UTC. Convert
  via `new Date((keepaTime + 21564000) * 60 * 1000)`.

---

## `resolve_codes`

Bulk UPC/EAN/GTIN/ISBN-13 → candidate-ASIN resolution for supplier
manifests (wholesale price lists, arbitrage CSVs). Accepts up to 500 rows
per call, batches the codes to Keepa, and attributes every returned
product back to your codes by scanning each product's full code list with
GTIN-14 normalization (the UPC-12 and EAN-13 forms of one item collide
correctly).

### What it's good for

- You have a supplier manifest (codes, maybe titles / brands / pack sizes)
  and need the candidate Amazon ASINs for each line.
- Identity only: no buy-box, offers, stock, or rating data. Feed the
  chosen ASINs into `get_product_details` for that.

### Token cost

- **~1 Keepa token per returned candidate** (typically ≈1 per code, since
  most codes match a single ASIN). `codeLimit` (default 3, max 20) caps
  candidates per code.
- Codes not in Keepa's database cost ~nothing. Worst case
  (`uniqueCodes × codeLimit`) is reserved up front and the unspent
  remainder refunded.

### Inputs

- **`rows`**: up to 500 manifest rows. Each row is a `code` plus optional
  `supplierTitle` / `supplierBrand` / `cost` / `qty` / `packSize`. Pass
  them when you have them: supplier title vs candidate title is the
  strongest disambiguator.
- **`domain`**: the marketplace to resolve on.
- **`codeLimit`**: candidates per code (default 3, max 20).
- **`excludeBrands`**: hard filter; candidates whose brand matches
  (case-insensitive) are dropped before storage.

### What it returns

A **compact summary only**: counts (`rows / resolved / multiCandidate /
notFound / invalid / unattributed`), tokens used, a `codes:` resultSetId,
the notFound code list, and per-row validation errors. **The per-row
candidate table is NOT returned here**: it is cached (24 h) and read via
`get_codes_result`. `multiCandidate` is your action signal: those rows
need disambiguation; single-candidate rows are done.

### Disambiguation: the tool states facts, you judge

It never auto-picks a winner. Each cached row carries the echoed supplier
inputs plus mechanical **matchSignals** (`candidateCount`,
`primaryCodeMatch`, `qtyMatch`, `brandMatch`) and per-candidate identity
fields (title, brand, full product codes, packageQuantity, BSR, lowest-new
price, `ambiguityFlags`). Compare and choose yourself.

### Notes

- Lenient validation (8–14 digits, formatting stripped). Placeholder /
  junk codes (≥10 identical consecutive digits) are rejected before spend.
- The resultSetId holds the **union of all candidates across rows**: for
  large manifests, disambiguate first and pass the chosen subset
  downstream, not the raw union (it can exceed a 500-ASIN cap).
- Rate-limited calls queue a `codes` background job: poll
  `check_job_status`; the completion message carries the resultSetId.

---

## `get_product_chart`

Fetches a PNG chart image from Keepa's `/graphimage` endpoint for a
single ASIN on a single marketplace domain. Renders inline in Claude
Desktop's regular chat and in Claude Code, and the model receives the
same image so it can analyse the chart and answer follow-up questions.

### What it's good for

- The user asks to **see** a price history, BSR trajectory, or Buy Box
  pattern.
- The user wants to **confirm a visual signal** (seasonality, trend
  direction) that a numeric field doesn't capture.
- A specific candidate from `get_product_details` or a finder result is
  worth a closer visual look.

Not a batch primitive, it's a visualization tool for specific products.
Don't ask for charts on every product in a result set.

### Token cost

**1 Keepa token reserved per call, flat.** Regardless of curves, image
size, or time range. Keepa caches identical requests **server-side for
90 minutes**: identical repeat requests on the Keepa side are free.

Note: the local token bucket doesn't reconcile against PNG responses
(they carry no `tokensConsumed` field), so the local available-tokens
count drops by 1 per call regardless of Keepa-side cache hits.

### Inputs

- **`asin`**: required. Single ASIN per call. Don't batch-chart a
  whole result set.
- **`domain`**: required. **No default.** If the user hasn't specified
  a marketplace, the assistant has to ask.
- **`rangeDays`**: defaults to 90 (last 90 days). Common values:
  30 (recent), 90 (typical sourcing window), 365 (seasonality).
- **`width` × `height`**: defaults to **800 × 400**.
- **Curve toggles**: see below.

### Curves

Tuned for resellers. **Default ON:** `amazon` (Amazon's own price),
`new` (3rd-party new offer low), `buyBox`, `salesRank` (BSR), `fba`
(lowest FBA offer).

**Default OFF (toggle on per user request):** `used`, `fbm`,
`buyBoxUsed`, `lightningDeals`, `warehouseDeals`, `primeExclusive`.

At least one curve must be enabled. Setting all 11 toggles to false is a
validation error: an empty chart would still cost 1 Keepa token for
nothing.

### What it returns

Every successful call returns the same two-part base:

1. **TextContent**, metadata line: ASIN, domain, range, dimensions,
   curves enabled, a Keepa product URL (browser fallback), and a
   token-cost note.
2. **ImageContent**: base64-encoded PNG (default 800 × 400). This is the
   channel the model sees (so it can analyse the chart), and it renders
   inline in Claude Code and Claude Desktop chat.

On Claude Desktop's regular chat, the server additionally mounts an MCP
Apps iframe to present the chart visually; every other surface uses the
image block above.

### Critical notes

- **Cowork limitation.** In Cowork (Claude Desktop's agent-mode surface)
  the chart can't be displayed: the sandboxed VM doesn't paint inline
  image content blocks and blocks reads of host-saved files. The model
  still receives the chart image for analysis, and the Keepa product URL
  in the text is the user-facing fallback: pair it with a
  `get_product_details` readout for the richest Cowork answer. This is a
  Cowork-side sandbox constraint, not an Agellic Lite bug. **Claude
  Desktop's regular chat and Claude Code render the chart inline.**
- **Non-PNG responses are rejected.** If Keepa returns an HTML error
  page with a 200 status, the tool surfaces `KEEPA_ERROR` instead of
  treating the bytes as an image.
- **API key stays server-side.** The URL with your Keepa API key is
  built in-process and never returned. Only rendered image bytes flow
  back.
- **Validation rejects fabricated ASINs** (`B0SNOW0001`-style
  sequences, `B00000001`, keyboard-mashed patterns) before hitting
  Keepa. Use ASINs from prior tool results or explicit user input.

---

## `check_token_balance`

Check Keepa token availability and per-tool cache-aware cost
projections. Free, reads locally cached bucket state and never calls
Keepa.

### What it's good for

- Before running an expensive call (a 50-ASIN deep dive, a large code
  resolution), check whether you'll fit synchronously or be queued to
  the background job runner. On base Keepa (1 TPM) that gate is where
  most non-trivial calls land, so quoting it up front sets the right
  expectation.
- Find out what fraction of a candidate set is already cached, so you
  can quote an accurate cost to the user instead of the worst-case.
- Get current balance and refill rate (tokens/min) for situational
  awareness.

### Token cost

**0 Keepa tokens.** Local bucket read.

### Inputs (all nullable)

Three call modes:

- **`asins` + `forTool` (+ `domain`)**: **Cache-aware per-tool check.**
  `forTool` is REQUIRED when `asins` is provided; on Lite the only
  per-ASIN-priced tool it covers is `get_product_details`, so that is
  the only accepted value. `domain` is REQUIRED in this branch, no
  silent default to US; if the user hasn't specified a marketplace, the
  assistant has to ask. Returns cached vs uncached counts, the actual
  per-tool cost, and a proceed / wait recommendation.
- **`estimatedCost`**: Simple affordability check. Returns whether
  enough tokens are available, or wait time.
- **All null**: Returns current balance and refill rate.

### What it returns

- Current token balance and refill rate (tokens/min).
- When `asins` provided: cached count, uncached count, actual cost,
  proceed/wait recommendation.
- When insufficient: tokens needed, estimated wait time in minutes.

### Per-tool cost reference

| Tool | Cost |
|------|------|
| `execute_keepa_finder` | 10 base + 1 per 100 ASINs; stats add 30 + 1/million `totalResults` |
| `get_product_details` (ASIN / resultSetId modes) | ~8 per uncached ASIN (9 reserved, graph delta refunded) |
| `resolve_codes` | ~1 per returned candidate, up to `codeLimit` (default 3, max 20) |
| `get_product_details` code lookup | 1 per returned candidate, up to `codeLimit` (default 3) |
| `get_product_chart` | 1 flat (cached by Keepa 90 min) |
| `get_finder_result`, `get_codes_result`, `check_job_status`, `check_token_balance` | free (local reads) |

### Timing expectations

- On base Keepa (1 TPM, 60-token bucket) a handful of uncached ASINs
  (roughly six deep-dive ASINs, or a small finder call) fit
  synchronously; the bucket refills at 1 token/min.
- Anything larger queues to the background runner and drains as tokens
  replenish. That is the normal path at 1 TPM, not an error state: poll
  with `check_job_status`.
- Cached re-reads are instant and free regardless of bucket state.

### Notes

- Cached ASINs cost 0 tokens for single-domain lookups: a
  partially-cached batch is quoted (and funded) at the lower actual
  price; only the uncached pulls are charged.
- On base Keepa (1 TPM) expect all but the smallest lookups to queue.
  A higher Keepa tier raises the bucket (capacity = tpm × 60) and pushes
  the sync ceiling up proportionally.

---

## `check_job_status`

Check the status of a previously-queued Keepa background job, or list
recent jobs. Used when a prior tool call returned a `jobId` because
Keepa rate limits forced the work onto the background queue.

On base Keepa (1 TPM, 60-token bucket) the queue is the normal path, not
an error: most calls beyond a handful of ASINs are funded over time
rather than up front, so this tool is where you watch a request's
**position in line**, the **token level it must reach** to start, its
**ETA**, and where you **cancel** a job you no longer want. It is a
first-class part of the Lite workflow, not an exception handler.

### What it's good for

- A prior tool call returned a `jobId`: poll with `action='status'`
  until completion, then re-call the original tool to fetch the result
  at 0 tokens from cache.
- Survey in-flight or recent jobs with `action='list'`.
- Cancel a queued job that hasn't started yet with `action='cancel'`.
- The user mentions a job without providing the ID: `list` first to
  discover the jobId, then proceed.

### Token cost

**0 Keepa tokens.** Local read against the job queue state.

### Actions

- **`list`**: Show all recent jobs with IDs, kinds, and statuses. No
  `jobId` required.
- **`status`** (default): Report the current status of a specific job:
  `pending`, `running`, `completed`, `cancelled`, or `failed`.
- **`cancel`**: Abandon a **pending** job that has not started yet
  (requires a `jobId`). A job already `running` is left to finish on its
  own: cancel only applies before execution begins.

### Status values

- **`pending`**: Waiting for tokens or a free runner slot. The status
  response reports the job's **position** in the queue, the **token
  level it must reach** before it can start, and a **refill-based ETA**.
- **`running`**: Job is currently executing.
- **`completed`**: Finished successfully.
- **`cancelled`**: Abandoned via `action='cancel'` before it started.
  Reported distinctly from `failed`, a cancellation is not an error.
- **`failed`**: Encountered an error. Status response carries the error
  message.

### What it returns

- **`list`**: Count of recent jobs plus one line per job: `jobId`,
  kind, status, attempt count, creation timestamp. A cancelled job is
  labelled cancelled, not failed.
- **`status`**: Status message. A `pending` reply includes the queue
  position, the token level the job is waiting for, and a refill-based
  ETA. **On `completed`, the message appends provenance guidance
  pointing at the retrieval tool + id needed to fetch the result at 0
  tokens.** The raw job payload is not echoed inline: treat the
  guidance as a fetch instruction. The retrieval hand-off looks like a
  `[handle:]` / `[partial:]` finder bracket for finder jobs (same
  convention as sync finder output), a `resultSetId` hand-off for
  lookup jobs, or a `codes:` resultSetId hand-off for code-resolution
  jobs: re-call the named tool with the named id to pull the actual
  result.
- **`cancel`**: Confirms the pending job was cancelled, or explains why
  it could not be (already running, already finished, or unknown id).

### Notes

- Jobs are created automatically when an enqueuing tool
  (`execute_keepa_finder`, `get_product_details`, `resolve_codes`) hits
  a Keepa rate-limit wait.
- Jobs expire after **24 hours**.
- Resubmitting an **identical** request while a matching job is still
  `pending` or `running` reuses that job instead of creating a duplicate
  that would double-spend tokens.
- If token replenishment is required for a `pending` job, the status
  surfaces its queue position, the token level it's waiting for, and a
  refill-based ETA, and the assistant offers refinement options (smaller
  subset, tighter filters, save for later).

---

## `get_finder_result`

Fetch a page of ASINs from a stored finder result set by id. Pairs with
`execute_keepa_finder` (which emits the id). Use `offset` and `limit` to
page through the full set without re-running the finder query.

### What it's good for

- After `execute_keepa_finder` returns a complete result set, retrieve
  the ASINs.
- Page through a large result set, or take a top-N slice. The slice can
  be passed to whichever downstream tool the user has chosen.
- The user asks for "next page" of results from a previous finder
  search.

### Token cost

**0 Keepa tokens.** Local read against the result-set cache.

### Inputs

- **`resultSetId`**: id from a prior `execute_keepa_finder` call.
- **`offset`**: defaults to 0. The starting index into the stored set.
- **`limit`**: defaults to 100. Number of ASINs to return.

### What it returns

A single human-readable text block:

- **Header line**: `Finder result <id> (domain <d>): returning K ASINs
  [offset A..B] of N stored (M total matches).` Carries: id echo, stored
  domain, count in this page, offset range, total count stored, and
  total finder matches.
- **ASIN line**: `ASINs: B001,B002,...` (comma-separated), or `ASINs:
  <none in range>` when `offset` is past the end.

### Notes

- **Result sets expire after 24 hours.** On "not found or expired",
  re-run `execute_keepa_finder` for a fresh id.
- **Pagination bounds.** If `offset >= totalResults`, the tool returns
  the header + `ASINs: <none in range>`, success with empty range, not
  an error.
- **Kind mismatch.** Only finder result sets work here. Passing a
  non-finder id (e.g. a `lookup:` or `codes:` id) returns an error
  pointing at the correct tool.
- **Malformed stored set (rare).** If a stored finder set lacks a
  recorded domain, the tool returns a "malformed: no domain recorded"
  error and the assistant re-runs the finder query.

---

## `get_codes_result`

The sole reader of cached `resolve_codes` resolutions. `resolve_codes`
returns only a summary; the per-row candidate table lives here: fetch
exactly the rows you need instead of receiving the whole table.

### What it's good for

- After `resolve_codes`, pull the rows that need disambiguation
  (`multiCandidateOnly: true`, the canonical move; single-candidate rows
  are already resolved).
- Fetch specific rows by supplier code, or page through the full set.
- Discover cached resolutions (`action: 'list'`).

### Token cost

**0 Keepa tokens.** Local cache read. Resolutions expire after 24 h: 
re-run `resolve_codes` to regenerate.

### Actions

- **`get`** (default): page through one resolution's rows. Requires `id`
  (`codes:...`).
- **`list`**: enumerate cached resolutions for this install, newest
  first, with summary counts. No id required.

### Inputs (get)

- **`id`**: the `codes:` id from `resolve_codes`.
- **`multiCandidateOnly: true`**, return only rows with >1 candidate.
- **`codes: [...]`**, fetch specific rows by supplier code (formatting
  and zero-pad variants are tolerated).
- **`offset` / `limit`**: row pagination over the filtered view (default
  50 rows/page).

### What it returns

Each row renders as a header line (the supplier code, the mechanical
matchSignals, the echoed supplier inputs, and an `excludedByBrand` count
when the filter dropped candidates) followed by a pipe table of
candidates: ASIN, title, brand, packageQuantity, items, part, model, BSR,
lowest-new price (cents), ambiguity flags (`variation` / `multipack` /
`inactive_listing`), and the primary-code match. Zero-candidate rows
render `(no candidates)`.

### Notes

- The price column is the current lowest-new price in cents, identity
  tier, so no buy-box data exists in this table by design.
- Compare the supplier title / brand / pack size in the header against
  each candidate and pick the ASIN yourself; the tool never auto-picks.
  Wildly heterogeneous candidates on one row signal a junk supplier code.
