# Usage examples

A starter set of prompts to try once Agellic Lite is installed and
connected. These are not exhaustive: Agellic Lite gives Claude nine
tools and they compose freely. This is a starter set; the per-tool
reference is in [TOOLS.md](./TOOLS.md).

The notes under each prompt describe roughly which tools Claude will
reach for and what the answer typically looks like. Token costs assume
nothing is cached; cached lookups are free for 24 hours. Agellic Lite
is free: the tokens are Keepa's, spent against your own Keepa
subscription.

---

## Discovery: finding products by criteria

When you don't have a list yet and want Claude to surface candidates
from Amazon's full catalog.

> Show me top-selling products in Toys & Games under $25 with at least
> 4 stars on the US marketplace.

Uses `execute_keepa_finder` with a category hint, price ceiling, and
rating floor. Claude returns a one-line summary like "Matched 1,847
products" and asks where you want to take it: narrow further, page
through the matches, deep-dive a top slice. ~11-30 tokens.

> Find Anker products under $40 with rating ≥ 4.2 and at most 5
> third-party sellers on the listing.

`execute_keepa_finder` with a brand filter plus competition gate. The
seller-count filter is the useful one here: it surfaces listings that
aren't already crowded. Typically returns a small enough set (tens to
low hundreds) to page through and deep-dive directly. ~20 tokens.

> Find kitchen products on Amazon US where the price has dropped at
> least $10 vs the 30-day average AND BSR is under 50,000.

`execute_keepa_finder` with `delta30_BUY_BOX_SHIPPING_gte` plus a BSR
ceiling. This is a recent-mover query: products that just got
cheaper and are still selling. ~20 tokens.

> Find pet supply products on the US marketplace that are Subscribe &
> Save eligible with rising review velocity over the last 30 days.

`execute_keepa_finder` with the S&S flag and a review-velocity delta.
Subscribe & Save eligibility is a recurring-revenue signal; combined
with accelerating reviews it points at items climbing the rank.
~15-25 tokens.

## Working through a finder result set

When a finder returned more matches than you want to look at at once and
you want to walk down to the products worth a close read.

> Page through those matches, 100 at a time. I'll tell you which slice
> to dig into.

`get_finder_result` reads the stored result set by id: free, no new
Keepa call. It hands back the ASINs in the requested range so you can
pick a slice. Result sets stay warm for 24 hours, so you can come back
to the same finder result in a later message without re-running (and
re-paying for) the query.

> Take the first 10 of those and give me full offer-level detail.

`get_product_details` on the shortlisted ASINs. Returns the full
per-product picture: individual seller offers, FBA vs FBM split, stock
depth, Buy Box rotation, calibrated demand range, observed sell-price
bands, seasonality confirmation, review velocity, OOS history, referral
fees, IP risk signals. Quoted at 16 tokens per uncached ASIN, settling
at ~6-8. On base Keepa (1 TPM) a 10-ASIN batch quotes 160 tokens, well
over an hour of refill, so Claude comes back with a **quote and asks you
to confirm** before anything starts. Say yes and it runs itself in the
background; you can read products as they settle without waiting for
the whole batch.

## Resolving supplier codes

When you have a supplier manifest of barcodes rather than ASINs.

> I have a supplier price list with 300 UPCs. Resolve them to Amazon
> ASINs on the US marketplace. [paste the codes]

`resolve_codes` bulk-resolves UPC / EAN / GTIN / ISBN codes (up to 500
rows per call) to candidate ASINs at Keepa's identity tier. Rows that
map to more than one candidate are flagged so you know which need a
human eye. Keepa bills ~1 token per returned candidate, but the order is
quoted at the worst case (`unique codes × 3`), so 300 codes quotes 900
tokens and comes back as a **confirmation request** on a base plan. It
settles far lower than the quote. Page the full per-row table any time
with `get_codes_result`, which is free, and read rows as they settle
mid-run with `check_job_status`.

> For the codes that resolved to a single ASIN, give me the calibrated
> demand estimate on each.

`get_product_details` on the clean matches. Identity resolution is
cheap; the demand and offer-level detail is where the tokens go. Quoted
at 16 tokens per uncached ASIN, settling at ~6-8.

## Deep-dive on a shortlist

When you have a handful of ASINs and need offer-level depth.

> Compare these 5 ASINs for Buy-Box stability over the last 90 days:
> B0CJT5D35W, B08N5WRWNW, B09G9FPHY6, B07FZ8S74R, B0ABC12345.

`get_product_details` returns each ASIN's Buy Box rotation table:
dominant seller, win share, unique winner count over the window, plus
a volatility flag in the insights block. Claude reads off who's
stable, who's getting flipped, and whether Amazon is in the mix. Quoted
at 80 tokens cold (5 × 16), so on a base plan expect a confirmation
request first; the actual charge lands nearer 30-40.

> For ASIN B0CJT5D35W, what's the calibrated monthly sales estimate
> and how confident is the model in it?

`get_product_details` on a single ASIN, which fits inline on a base plan.
The `demand` block leads with a
`mode` field that tells you which estimation path fired: `read` (a
computed range with low / likely / high plus a confidence label of
`high`/`medium`/`low`), `badge` (Amazon's own "X+ bought past month"
figure, taken as ground truth), or `no-read` (signal too weak to
estimate, with a reason). This is a range estimate, not a forecast: the
confidence and mode matter as much as the number. 16 tokens quoted,
~6-8 charged.

> Look up the ASIN for UPC 012345678905 on Amazon US.

`get_product_details` in code-lookup mode, for a one-off code. Returns
up to 3 candidate ASINs with title + brand for disambiguation. 3
tokens. This is identification only: no offers / stock / rating data;
for that, take the resolved ASIN and call again with `asins`. For a
whole manifest of codes at once, reach for `resolve_codes` instead.

## Price and BSR history

When you want to see the picture, not the numbers.

> What does the price and BSR history of B0CJT5D35W look like over the
> last 90 days?

`get_product_chart` returns a PNG with the default reseller view:
BSR, lowest new price, Buy Box price, and lowest FBA offer, all
overlaid on a single chart at 800×400. 1 token. Claude Desktop
(regular chat and Cowork), Claude Code, and the ChatGPT desktop app
(with the opt-in MCP Apps flag, see INSTALL.md) all render the chart
inline.

> Pull the BSR-only chart for the same ASIN at 365 days. I want to
> see if there's seasonality.

Same tool, `rangeDays: 365`, other curves toggled off. The longer
window makes annual patterns (Q4 ramps, summer dips, back-to-school
spikes) visible. 1 token.

## Market sizing and insights

When the question is about the shape of a market, not specific
products.

> I want to size the Anker accessories market on Amazon US: products
> under $40 with BSR under 100,000. What's the average Buy Box price,
> how many sellers per listing, and how often is Amazon on these
> listings?

Two-step, and on Agellic Lite the two steps are mandatory. First call:
`execute_keepa_finder` filters-only to learn the match count. Second
call: the same filters plus `includeStats=true` **and**
`expectedTotalResults` set to that count, which is what prices the
surcharge before anything is spent (without it the call is refused, with
nothing created and nothing charged). You get the `searchInsights`
summary: average landed Buy Box price (item + shipping, the same amount
Amazon charges its referral fee on), median seller count, the share of
listings where Amazon is the Buy Box winner, brand fragmentation across
the match set, FBA share, average rating, average review count. The
stats leg costs `30 + ⌊totalResults / 1M⌋` tokens, so under a million
matches it is exactly 30. On a 60-token bucket that is most of your
hour, and Claude will say so and ask before spending it.

> In Pet Supplies on US, what's the typical out-of-stock rate and
> seller count on listings with BSR under 25,000?

Same pattern: finder filters narrow it down, then `includeStats`
reads the shape. Useful before you commit to sourcing in a category
you haven't worked before: high OOS rates and thin seller counts mean
opportunity; low OOS and 20+ sellers per listing usually mean the
category is saturated.

---

## Things to know before you start

- **Tokens are Keepa's currency, and Agellic Lite is free.** You pay
  only Keepa, on your own subscription. Base Keepa refills 1 token per
  minute, so long runs become background **work orders** when the bucket
  runs low. That's normal at 1 TPM, not a fault: Claude polls with
  `check_job_status`, and the order funds itself as tokens refill. Run
  `check_token_balance` any time to see what you have.
- **Big work asks first.** Anything quoted above 60 minutes of refill
  (on a base plan, roughly 60 tokens) stops and shows you a quote plus a
  time bound instead of starting. Nothing is charged until you agree.
  Expect this on a deep-dive of four or more uncached products, or a
  code manifest over about twenty rows.
- **Cost and time are two separate claims.** The cost is a flat
  worst-case ceiling for your order. The time is a queue-aware bound,
  "done within ~X of token refill", that is recomputed on every poll and
  counts down. It assumes nothing else is spending your Keepa key, and
  its wall-clock figure assumes a session open about 8 hours a day.
- **Work orders run on your machine, not the cloud.** An accepted
  product-details batch or code resolution advances as tokens refill,
  but only while a connected app stays open. Quit Claude (or let the
  machine sleep) and it pauses; relaunch and it resumes where it left
  off, nothing lost. Kick off the big one, leave Claude running, come
  back to results.
- **You can read a long order before it finishes.** Ask for what has
  settled so far and Claude pages it with `check_job_status`
  `action: "fetch"`. You can also stop an order at any point with
  `action: "cancel"`: it halts at the next dispatch boundary and keeps
  everything already settled.
- **Don't ask Claude to "retry" a running order.** There is no
  deduplication, so a re-run pays Keepa for the same work twice. Poll the
  order instead.
- **The 24-hour cache is real.** Re-asking the same question later the
  same day is usually free, and the cache is shared across chats and
  both Claude apps, so a fresh conversation re-reads it for nothing.
- **Stored results stay warm.** A finder result set or a code-resolution
  table is retrievable by id for 24 hours, so you can page back into it
  in a later message with `get_finder_result` / `get_codes_result`
  without re-running (and re-paying for) the original query. If the view
  has expired, the order behind it is kept for 14 days and
  `check_job_status` rebuilds the view for free.
- **Claude won't auto-chain expensive steps.** A finder result won't
  silently roll into a large product-details batch. Claude waits for you
  to ask. Compound asks in a single turn ("find X and deep-dive the top
  10") work as expected.
- **Triage before you enrich.** At 1 token per minute the cheap tools do
  the sorting: a `perPage=100` finder round is 11 tokens and a chart is
  1, against 16 quoted for a single enriched product. A discover, chart,
  then deep-read pass on two survivors fits inside one hour's refill;
  enriching four products blind does not.
