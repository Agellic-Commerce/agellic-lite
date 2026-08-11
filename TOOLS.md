# Tool Reference

This document is the practitioner reference for the 9 MCP tools exposed by
Agellic Lite v2.0.0, the free, bring-your-own-Keepa-key edition of the
Agellic MCP server. Each section covers what the tool does, what it costs
in Keepa tokens, what inputs it accepts, what it returns, and the operating
rules worth knowing before you turn it loose on a candidate set. All Keepa
token costs are concrete numbers measured against current runtime behavior,
no hedging.

Agellic Lite pulls every figure from Keepa (`keepa.com`) and your Keepa
subscription supplies the token bucket. If you don't have a Keepa account
yet, grab one at [keepa.com/#!api](https://keepa.com/#!api) before you
start. Base Keepa gives you **1 token per minute** (a **60-token bucket**,
since capacity = tpm × 60). At that rate background execution is the normal
path for anything but the smallest calls, not an error state: every
Keepa-calling tool turns your request into a durable **work order** that
funds itself as tokens refill, and a work order survives restarts (quitting
pauses it, relaunching resumes it). A free `/token` boot probe auto-detects
your key's real rate at startup, so a higher Keepa tier unlocks a
proportionally larger bucket without hand-tuning. Several sections below
note where the 1-TPM bucket is the binding constraint for inline vs
background behavior.

## Quick scan

| Tool | One-line summary | Token cost |
|------|------------------|------------|
| `execute_keepa_finder` | Discover ASINs by category / brand / price / rank / competition filters; optional market-insights stats. | 10 base + 1 per started 100 ASINs returned (stats: +30 base + 1 per whole million total matches) |
| `get_product_details` | Deep per-product analysis: offers, Buy Box rotation, stock depth, calibrated demand, insights. Also resolves UPC/EAN/ISBN → ASIN. | 16 quoted per uncached ASIN, settling at ~6-8 (code lookup: 1 per candidate up to `codeLimit`) |
| `resolve_codes` | Bulk-resolve supplier UPC/EAN/GTIN/ISBN codes (up to 500 rows) to candidate ASINs at the identity tier. | quoted `uniqueCodes × codeLimit`, settling at ~1 per returned candidate |
| `get_product_chart` | Fetch a price/BSR history PNG chart for one ASIN on one marketplace. | 1 flat per call (Keepa 90-min server cache) |
| `check_token_balance` | Show current Keepa token balance, refill rate, and cache-aware per-tool cost projections. | 0 (local) |
| `check_job_status` | The work-order hub: list orders, read one order's status and live ETA, page settled rows mid-run, or cancel. | 0 (local) |
| `confirm_work_order` | Authorise a quoted work order that is too big to start without asking. | 0 (local) |
| `get_finder_result` | Page through ASINs from a stored finder result set by id. | 0 (local) |
| `get_codes_result` | Page through a stored code-resolution result set (the per-row candidate table). | 0 (local) |

The 5 free tools (`check_token_balance`, `check_job_status`,
`confirm_work_order`, `get_finder_result`, `get_codes_result`) read local
state only and never charge Keepa tokens. Cached re-reads of the paid tools
are also free for 24 hours.

Domain values referenced throughout: `1`=US, `2`=UK, `3`=DE, `4`=FR,
`5`=JP, `6`=CA, `8`=IT, `9`=ES, `10`=IN, `11`=MX, `12`=BR.

---

## Work orders: the three answers a call can give

`execute_keepa_finder`, `get_product_details`, and `resolve_codes` do not
fail on a rate limit and do not return a `jobId`. Each call seals your
request as a durable **work order** with a `wo_...` id, prices it, and
returns exactly one of three answers.

1. **Inline.** The balance covers the quote, so the work runs in the call
   and you get the normal result. There is an order behind it; you only
   hear the id if the run stopped short, in which case the reply opens
   `PARTIAL` (resumable, keep polling), or `INCOMPLETE` / `CANCELLED`
   (final, what you see is everything).
2. **Background acceptance.** The balance is short, so the call is accepted
   anyway. The reply carries the `orderId` and two separate claims: a flat
   `Cost:` line (worst-case tokens at the quoted rate, no time claim), then
   a `Queue:` / `ETA:` pair naming how many orders are ahead and a **bound**
   on the wait, "done within ~X of token refill".
3. **A consent quote.** The work needs more than **60 minutes of token
   refill** (on a base plan at 1 TPM, a quote above roughly 60 tokens), so
   nothing starts and nothing is charged. The reply carries the `orderId`,
   a one-time `confirmToken`, the `Quote:` line, and the same Queue and ETA
   pair framed "If you confirm now:". Start it with `confirm_work_order`.

`execute_keepa_finder` never produces a consent quote: one query is one
indivisible Keepa call, so it discloses cost and ETA as it accepts. A finder
query priced above what the bucket could ever hold at once is refused at the
door instead, with nothing created and nothing charged.

**About that ETA.** It is a bound, not a prediction, recomputed fresh on
every poll so it counts down as the queue drains and the balance refills.
It bounds the work **as currently quoted** (an order ahead can outgrow its
own quote), it assumes **no other work on this Keepa key**, and its
wall-clock figure assumes a session open **about 8 hours a day** (an
always-open session finishes roughly 3× sooner). Work advances only while a
connected app is open, and polling neither speeds an order up nor changes
its ETA.

`get_product_chart` is the one paid tool that is not a work order: it costs
a flat token and, on an empty bucket, simply asks you to retry in a few
seconds rather than creating an order.

Everything after an order exists happens through `check_job_status`: watch
it, read its settled rows mid-run, or stop it. See
[TOKENS-AND-QUEUE.md](./TOKENS-AND-QUEUE.md) for the fuller picture.

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
  follow-up call after a filter-only recon to get avg landed Buy Box
  price (`avgBuyBox`: item + shipping, the same amount Amazon charges
  its referral fee on), seller counts, Amazon share, brand
  fragmentation, FBA share, avg rating, and avg review count across the
  matched set.

Not the right tool when the user already has ASINs: go straight to
`get_product_details`.

### Token cost

- **Base: 10 tokens** + **1 token per started 100 ASINs returned**.
- **Stats (`includeStats=true`):** +30 base + 1 per **whole completed**
  million of `totalResults` (`⌊totalResults / 1,000,000⌋`, so under 1M
  matches it is exactly the flat 30).
- `perPage` is a **ceiling on cost, not a fixed charge**. Keepa bills
  `10 + ⌈min(perPage, totalResults)/100⌉`. So `perPage=10000` on a
  4,231-match query costs **53 tokens**, not 110. Even the worst case
  (`perPage=10000` returning a full 10,000 ASINs) is 110 tokens.
- A broad **single-filter** search can match hundreds of millions of
  products and cost 300+ tokens. Always combine ≥ 2 filters.

### `expectedTotalResults` is required for stats on Agellic Lite

`includeStats=true` without `expectedTotalResults` is a front-door error
here (nothing created, nothing charged). The stats surcharge rides the match
count, which nobody knows before the call returns, and an unbounded request
can overdraw hours of refill on a 60-token bucket. The two-step shape is
therefore mandatory on Lite:

1. Run the filters once with `includeStats` omitted (cheap) and read
   `totalResults`.
2. Re-run the same filters with `includeStats: true` and
   `expectedTotalResults: <that number>`, so the surcharge is priced before
   anything is spent.

`expectedTotalResults` is a **local** pricing input: it is used to quote the
call and never sent to Keepa. Keepa always bills on the real match count, so
the bound constrains the quote, not the charge.

**Price insights against the whole bucket, not against the call.** At 1
token per minute a bounded stats call at `perPage=100` costs about 41
tokens, roughly two-thirds of your hour, against 11 tokens for another
filters-only refinement round. Nothing refuses the call, but when the stats
re-run would be more than half the bucket the filters-only response says so
in tokens, share of bucket, and minutes of refill, and asks first.

### Inputs

All filters are ANDed; array filters are ORed (max 50 entries each). At
least one real filter is required besides `domain` / `page` / `perPage`
/ `sort` / `stats`.

Common numeric range filters (use `_gte` / `_lte` suffixes):

- `current_BUY_BOX_SHIPPING`: Landed Buy Box price, item + shipping, in
  cents ($40 → 4000). The default when the user says "price".
- `current_SALES`: Best Sellers Rank (lower = better).
- `current_RATING`: 0–50 scale (4.5 stars → 45).
- `current_COUNT_NEW`: Listed new-seller count. Default when the user
  says "sellers".
- `delta30_BUY_BOX_SHIPPING`, `delta90_BUY_BOX_SHIPPING`: Landed Buy Box
  price delta vs avg (positive = price dropped).
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

- `buyBoxStandardDeviation30/90/365`: Landed Buy Box price volatility in
  cents. `_lte` for stable, `_gte` for volatile.
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
- **Not funded yet:** no ASINs at all, just an `orderId` (`wo_...`), the
  cost, and a queue-aware ETA bound. The query runs itself as tokens
  refill; poll with `check_job_status`, and the handle arrives through
  `status` or `fetch` once it lands. Do not re-call the finder to "retry",
  that pays for the same query twice.
- **Refused at the door:** the quote is larger than this bucket could ever
  make available to one query. An error naming the cost and the cheaper
  query to run (a lower `perPage`, or dropping `includeStats`, or tighter
  filters when the per-million surcharge is what blows the ceiling).
  Nothing is created and nothing is charged. A refill rate of 0/min is
  refused the same way.
- **Zero results:** plain-text message including the actual charge (a
  zero-match query still bills); the model suggests filter adjustments.

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

- **16 tokens per uncached ASIN, quoted.** That is the measured worst case
  a lookup order authorises per row (two 6-token offer pages, stock,
  rating, graph). Each dispatch reserves that ceiling and settles at
  Keepa's own figure, so **actuals land at ~6-8**. Read the quote as the
  ceiling being authorised and the ledger figure as the cost.
- **0 tokens for cached ASINs** (24h product cache). They are excluded from
  the quote entirely.
- Partial cache hits: only uncached ASINs are quoted and charged.
- **On base Keepa the consent gate fires early.** At 1 TPM, 60 minutes of
  refill is 60 tokens, so **four or more uncached ASINs** quote past the
  threshold and come back as a consent quote rather than starting. Three
  uncached ASINs (48 tokens) still run inline on a healthy bucket.

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

Every successful fresh fetch persists a `lookup:<orderId>` result set, so a
follow-up call can re-read the same ASINs by id at 0 tokens.

### What it returns

Per resolved ASIN:

- **`identity`**: title, brand, category, manufacturer, `productCodes`
  (UPC/EAN/GTIN), Amazon URL, image URL.
- **`pricing`**: current lowest new price, list price, Buy Box price,
  30/90/180d averages, trend direction/strength, volatility score, and
  the **sell-price read** (`pricing.sellPrice`): observed sale-price
  bands (`moveFastCents` / `marketCents` / `stretchCents` = p25 / median
  / p75 of prices in force at inferred sale moments), the current
  price's position inside them, sales skew, drift, and caveats. Bands
  describe the observed market, never a recommendation.

  Every Buy Box price in this block is **landed** (item + shipping),
  Keepa's native `csv[18]` basis: `pricing.buyBox.currentCents`, the
  30/90/180/365d averages, and the 365d min/max.
  `pricing.buyBox.shippingBasis: 'landed'` marks that contract, and its
  absence marks a stale pre-landed payload. When shipping is greater
  than zero, `itemCents` and `shippingCents` show how the landed price
  splits; on free-shipping items both are absent.
  `pricing.sellPrice` declares its own `shippingBasis` (`landed`,
  `item-only`, or `mixed-window`) and carries
  `marketState: 'suppressed'` when the Buy Box is suppressed.
- **`sales`**: current and historical sales rank, primary + leaf BSR
  with category names, 30/90/180d drops, Amazon-reported monthly sold
  badge (when available; the model's range estimate lives in `demand`).
- **`demand`**: calibrated monthly-sales range (`rangeLow` /
  `centerLikely` / `rangeHigh`), confidence (`high` / `medium` /
  `low`), and `mode` (`read` / `badge` / `no-read`, the discriminant
  that gates which other fields are present, read it first), plus
  `caveats` (free-text qualifiers like "estimate from cross-marketplace
  baseline").
- **`seasonality`**: whether a recurring seasonal peak exists and how
  sure the detector is. `confirmationLevel` separates `confirmed` (the
  peak recurred across 2+ years) from `candidate` (one season observed,
  do not act on it alone); also the peak week window, calendar label
  (Q4, Summer, etc.), current phase (`pre-peak` / `peak` / `post-peak` /
  `off-season`), and concrete sourcing-window / lead-out dates.
- **`competition`**: individual seller offers (FBA/FBM, prices, stock
  depth), Buy Box current winner, dominant seller + win %, rotation
  table, historical avg seller count. Offers are ordered by **landed**
  price, so the offer at the top is the cheapest to actually receive,
  not the one with the lowest sticker and a shipping charge behind it.
  `competition.buyBox.priceCents` is the same landed value and basis as
  `pricing.buyBox.currentCents`.
- **`reviews`**: rating, review count, velocity (added 30/90/180d),
  trend (accelerating/steady/slowing), historical avg.
- **`supply`**: Amazon OOS 30/90/180d, marketplace OOS 90d.
- **`insights`**: rank volatility, trend signals, Buy Box volatility,
  effective competition (sellers within 5% of BB), IP risk,
  race-to-bottom warning. See
  [`COMPUTED-INSIGHTS.md`](COMPUTED-INSIGHTS.md) for the algorithms
  behind every field in `demand`, `seasonality`, `pricing.sellPrice`,
  and `insights`, what each measures, the constants, and how to read
  it.
- **`economics`**: referral fee percent, FBA pick & pack fee, return
  rate.
- **`metadata`**: listing age, Subscribe & Save eligibility.
- **`tokensUsed`**: actual tokens consumed by this call.

When the balance is short, the reply carries an `orderId` and no products:
poll with `check_job_status`, read products as they settle with
`action: "fetch"`, and once the order finishes a re-call with the same
`asins` / `resultSetId` serves cached data at 0 tokens.

### Cost discipline

State the worst-case spend before running on a stored result set
(`uncached_asins × 16` tokens, with actuals typically ~6-8; the cache
absorbs anything fetched in the last 24h). Per-product output is **~5–10 KB**
depending on offer count (up to 20 offers per product) and insight
verbosity. 10 ASINs ≈ ~75 KB / ~20K tokens of context; 50 ASINs ≈ ~375 KB /
~100K tokens. There's no fixed "batch size target": state the spend, then
let the user decide the batch size. On base Keepa (1 TPM, 60-token bucket)
three uncached ASINs is the inline ceiling, so anything larger becomes a
background order and, from four ASINs up, a consent quote first.

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

- **Quoted at `uniqueCodes × codeLimit`, settled at Keepa's actual charge**
  as each dispatch commits. There is no preflight reservation to refund;
  the ledger is what carries the honest cost.
- Keepa bills **~1 token per returned candidate** (typically ≈1 per code,
  since most codes match a single ASIN). `codeLimit` (default 3, max 20)
  caps candidates per code. Codes not in Keepa's database cost ~nothing.
- **On base Keepa the quote crosses the consent threshold fast.** At the
  default `codeLimit: 3` and 1 TPM, a manifest of more than about 20 unique
  codes quotes past 60 minutes of refill and comes back as a consent quote.
  A 300-code manifest quotes 900 tokens (15 hours of refill) and settles far
  lower, but you are asked before it starts.
- There is no cache discount: this identity tier never writes the product
  cache, and the question a code asks is *which* ASINs it maps to.

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

The `codes:` id is derived from the work order and stable for its whole
life, so an id handed to you mid-run still resolves against whatever the
order ends up with. If a run stopped short, the reply opens with a note
before the summary: `PARTIAL` / `NOT STARTED` means the order is alive and
resumable (poll the `orderId`), while `INCOMPLETE` / `CANCELLED` means it is
final.

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
- **Zero candidates does not always mean "not found".** A code the run
  never reached renders exactly like a genuine Keepa miss. The summary's
  `PARTIAL:` line is what tells the two apart, so confirm with
  `check_job_status` before treating a `notFound` code as dead.
- A call the bucket cannot fund is accepted as a background order, not
  refused: poll `check_job_status`, which hands back the `codes:`
  resultSetId once anything has settled.

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
   curves enabled, and a token-cost note.
2. **ImageContent**: base64-encoded PNG (default 800 × 400). This is the
   channel the model sees (so it can analyse the chart), and it renders
   inline in Claude Code and Claude Desktop chat.

On Claude Desktop (regular chat and Cowork) the server additionally
mounts an MCP Apps view to present the chart. In the ChatGPT desktop
app the chart renders as an inline MCP Apps card when Codex's
`enable_mcp_apps` feature flag is on (the installer offers to set it;
see [INSTALL.md](./INSTALL.md#inline-chart-cards-in-chatgpt-desktop-optional))
and you are signed in to ChatGPT. The Codex CLI terminal does not
render images; the model still receives the chart for analysis.

### Critical notes

- **Cowork renders inline as of v1.7.1.** Earlier releases could not
  display the chart in Cowork (Claude Desktop's agent-mode surface);
  it now arrives as the same MCP Apps view regular chat uses. If
  Cowork still shows text-only charts after an upgrade, quit Claude
  Desktop fully (Cmd-Q) and reopen: Cowork keeps the previous server
  process alive until a full quit.
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
  resolution), check whether it will run inline or become a background
  work order. On base Keepa (1 TPM) that gate is where most non-trivial
  calls land, so quoting it up front sets the right expectation.
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
| `execute_keepa_finder` | 10 base + 1 per started 100 ASINs; stats add 30 + 1 per whole million of `totalResults` |
| `get_product_details` (ASIN / resultSetId modes) | **16 per uncached ASIN** quoted (the authorised worst case), settling at ~6-8 |
| `resolve_codes` | quoted `uniqueCodes × codeLimit` (default 3, max 20), settling at ~1 per returned candidate |
| `get_product_details` code lookup | 1 per returned candidate, up to `codeLimit` (default 3) |
| `get_product_chart` | 1 flat (cached by Keepa 90 min) |
| `get_finder_result`, `get_codes_result`, `check_job_status`, `confirm_work_order`, `check_token_balance` | free (local reads) |

This tool's per-ASIN estimate for `get_product_details` is the **authorised
ceiling**, the same 16 the order quotes from, so an estimate that fits the
balance runs inline. Quote the spend to the user at the ~6-8 figure, and
remember that an unaffordable estimate is not a refusal: the order still
runs, just in the background.

### Timing expectations

- On base Keepa (1 TPM, 60-token bucket) a small finder call, a chart, or
  up to three uncached deep-dive ASINs fit inline; the bucket refills at 1
  token/min.
- Anything larger becomes a background work order and funds itself as
  tokens replenish. That is the normal path at 1 TPM, not an error state:
  poll with `check_job_status`.
- Cached re-reads are instant and free regardless of bucket state.

### Notes

- Cached ASINs cost 0 tokens for single-domain lookups: a
  partially-cached batch is quoted (and funded) at the lower actual
  price; only the uncached pulls are charged.
- On base Keepa (1 TPM) expect all but the smallest lookups to run in the
  background. A higher Keepa tier raises the bucket (capacity = tpm × 60)
  and pushes the inline ceiling up proportionally.
- **The `Wait ~N minutes` figure here is not an order's real wait.** It is
  a hypothetical balance-over-refill-rate calculation for a call made right
  now, and it cannot see other work already queued on this key. Once a call
  has created a work order, `check_job_status` reports the real,
  queue-aware ETA, which recomputes on every poll.
- **A 0/min refill rate is a hard refusal, not a wait.** Keepa product
  tracking consumes refill rate; the fix is to shrink the tracked-product
  list or upgrade the plan, and a full bucket does not rescue it.

---

## `check_job_status`

The hub for every work order: watch one, list them all, read the rows one
has already settled, or stop it. Every id it accepts is a `wo_...` work
order, created by `get_product_details`, `execute_keepa_finder`, or
`resolve_codes` on every call they accept.

The tool name and the `jobId` input field are historical, left over from a
background job queue that no longer exists. Pass a work-order id.

On base Keepa (1 TPM, 60-token bucket) background execution is the normal
path, not an error: most calls beyond a handful of ASINs are funded over
time rather than up front, so this is where you watch **how many orders are
ahead**, the **live ETA bound**, whether the order has **actually started**,
and where you read or stop it. It is a first-class part of the Lite
workflow, not an exception handler.

### What it's good for

- A prior tool call returned an `orderId`: poll with `action='status'`
  (pass it as `jobId`).
- An order is running and you want the rows it has so far:
  `action='fetch'`. You do not have to wait for it to finish.
- Survey in-flight or recent work with `action='list'`.
- The user mentions an order without providing the id: `list` first to
  discover it, then proceed.

### Token cost

**0 Keepa tokens.** Local read; never calls Keepa. Polling costs nothing
and speeds nothing up.

### Actions

- **`list`**: every recent order, newest first, one line each: id, kind,
  state, rows done of total, tokens charged, creation timestamp. No
  `jobId` required.
- **`status`** (default): the full report on one order (see below).
- **`fetch`**: page through the rows the order has already settled,
  mid-run or after it finishes, rendered the way the creating tool renders
  them: a deep-dive order as the same product blocks `get_product_details`
  prints, a finder order as a one-line match count plus the handle its
  ASINs are cached under, a codes order as one line per settled code.
  `offset` (default 0) and `limit` (default 100, max 500) page in request
  order, so a page means the same thing on every call.
- **`cancel`**: stop it. An order that has **not started** cancels
  instantly. A **running** one accepts the request and stops at its next
  dispatch boundary (a Keepa request already in the air is paid for either
  way), and everything it settled stays readable with `fetch`.

### What `status` reports

- **State**, and for a `quoted` order the live `confirmToken`.
- **Items**: rows settled of total, split into ok / not-found / error, plus
  how many are still pending.
- **Tokens charged**, separated into what Keepa itself billed and what is
  reserved for requests still in flight. That reserve releases when they
  settle, so a mid-run charged total is a **high-water mark, not a bill**,
  and it can go down.
- **Quote**: the original worst-case cost at the rate it was quoted at.
  Cost only, no time claim.
- **Projected**, and **DRIFT** when the work still remaining now needs more
  refill than the whole order was quoted at (cached rows expiring mid-run
  and a falling refill rate both do this). The quote is never retroactively
  rewritten.
- **Queue and ETA**: how many orders are ahead with what they still have to
  spend, then a bound on the wait. Recomputed fresh on every poll, so it
  counts down.
- **`authorised:`** and **`started:`**, which are different events.
  `authorised:` is when the work was agreed to; `started:` says only
  whether a Keepa request has actually been dispatched. An order still
  waiting reads `not yet started`, naming how many orders are ahead.
- A **hand-off line** naming the result-set id and the free tool that reads
  it, once anything has settled.

### Work-order states

- **`draft`**: still being staged. Not priced, not started.
- **`quoted`**: priced and waiting on your consent. Nothing charged.
- **`confirmed`**: authorised, which is not the same as started. It may sit
  behind other authorised orders before its first Keepa request, and it
  advances only while a session is open.
- **`completed`**: every row settled.
- **`partial`**: stopped with rows unfetched (a spend ceiling, a deadline,
  an exhausted retry budget, or repeated upstream refusals) with everything
  it did settle intact. Read them with `fetch`.
- **`failed`**: stopped before settling anything.
- **`cancelled`**: stopped on request, keeping its settled rows.

### When the ETA is withheld

The Queue and ETA pair is **withheld entirely** for four states, because
none of them finishes on refill alone and a completion time would
contradict the message it sits in: a `draft`, an order asked to cancel, an
order whose commit log is unreadable, and one whose deadline has passed.
For an order still awaiting consent the pair is kept but framed
conditionally, "If you confirm now:".

### Notes

- **There is no deduplication.** Re-calling a tool to "retry" a background
  order creates a second order and pays Keepa twice. Poll the id you have.
- Finished work orders are kept for **14 days**. Their result-set view
  holds a 24-hour TTL and is rebuilt on demand from the order's own record
  if it lapsed, at 0 tokens.
- An order waiting on refill holds **no** token reservation; nothing is
  held until a dispatch actually fires. One still `quoted` has been charged
  nothing.
- If token replenishment is needed before an order can advance, the
  assistant relays the wait briefly and offers refinement options (smaller
  subset, tighter filters, save for later).

---

## `confirm_work_order`

The consent gate. When a tool quotes work costing more than **60 minutes of
Keepa token refill**, it does not start it: it returns an `orderId`, a
cost-only quote, a separate ETA bound, and a one-time `confirmToken`. Pass
the id and token back here to authorise it.

On base Keepa (1 TPM) this gate is easy to reach: 60 minutes of refill is
60 tokens, so four or more uncached ASINs on `get_product_details`, or a
manifest of more than about twenty codes on `resolve_codes`, lands here.
`execute_keepa_finder` never does, and never produces a `confirmToken`.

### What it's good for

- Starting a quoted order after the user has seen the numbers and agreed.

### Token cost

**0 Keepa tokens.** It reads local state and writes a consent record; the
Keepa spend happens later, as the order runs.

### Inputs

- **`orderId`**: the id returned alongside the quote (`wo_<timestamp>_<suffix>`).
- **`confirmToken`**: the one-time token issued with that quote. It is
  bound to that quote, and any re-quote invalidates it.

### What happens

| Order state | Result |
|---|---|
| `quoted`, rate steady | **Authorised and queued** for the background scheduler, which is not the same as started. It runs behind anything already queued; `check_job_status` reports whether it has actually started. |
| `quoted`, rate dropped >20% | **Re-quoted.** A new quote, a new ETA, and a new token come back and nothing starts. This is a success, not an error: show the new numbers and ask again. The old token is dead. |
| `quoted`, rate now 0/min | Refused with an explanation. Your token stays valid, since the rate can recover. |
| `draft` | Error: never sealed, its manifest is still open. |
| `confirmed` | Error: already authorised. Use `check_job_status`. |
| terminal | Error: already finished. Read its rows with `check_job_status` `action: 'fetch'`. |
| wrong/expired token | Error: re-read the current quote with `check_job_status`, which reports the live token. |

### Critical notes

- **Nothing is charged at confirmation time.** Tokens are spent per batch
  as the order executes, and every batch is recorded, so a partially-run
  order's cost is always exactly what it consumed.
- **Confirming queues it, it does not start it.** Only the `started:` line
  on `check_job_status` tells you dispatch has begun.
- **A confirmed order can still be stopped** with `check_job_status`
  `action: "cancel"`. It stops at its next dispatch boundary and everything
  settled stays readable.
- This tool never runs work inline and returns no rows at all. Read the
  work with `check_job_status`, whichever tool quoted it.

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

- **The stored view expires after 24 hours, but the order does not.** A
  finder result set is keyed `finder:<orderId>`. On "not found or expired",
  poll `check_job_status({ jobId: "<orderId>", action: "status" })` first:
  it rebuilds the view from the order's own record at 0 tokens. Re-running
  the finder would pay Keepa a second time.
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

**0 Keepa tokens.** Local cache read. The stored view expires after 24 h,
but the work order behind it is kept for 14 days: if the id is gone, poll
`check_job_status({ jobId: "<orderId>", action: "status" })` to rebuild the
view at 0 tokens rather than re-running (and re-paying for) the resolution.

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
