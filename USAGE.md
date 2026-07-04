# Usage examples

A starter set of prompts to try once Agellic Lite is installed and
connected. These are not exhaustive: Agellic Lite gives Claude eight
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
depth, Buy Box rotation, calibrated demand range, review velocity, OOS
history, referral fees, IP risk signals. ~8 tokens per uncached ASIN. On
base Keepa (1 TPM) a 10-ASIN batch (~80 tokens) is larger than one
bucket, so it queues as a background job and Claude polls until done.

## Resolving supplier codes

When you have a supplier manifest of barcodes rather than ASINs.

> I have a supplier price list with 300 UPCs. Resolve them to Amazon
> ASINs on the US marketplace. [paste the codes]

`resolve_codes` bulk-resolves UPC / EAN / GTIN / ISBN codes (up to 500
rows per call) to candidate ASINs at Keepa's identity tier. Rows that
map to more than one candidate are flagged so you know which need a
human eye. ~1 token per returned candidate. Page the full per-row table
any time with `get_codes_result`, which is free.

> For the codes that resolved to a single ASIN, give me the calibrated
> demand estimate on each.

`get_product_details` on the clean matches. Identity resolution is
cheap; the demand and offer-level detail is where the tokens go. ~8
tokens per uncached ASIN.

## Deep-dive on a shortlist

When you have a handful of ASINs and need offer-level depth.

> Compare these 5 ASINs for Buy-Box stability over the last 90 days:
> B0CJT5D35W, B08N5WRWNW, B09G9FPHY6, B07FZ8S74R, B0ABC12345.

`get_product_details` returns each ASIN's Buy Box rotation table:
dominant seller, win share, unique winner count over the window, plus
a volatility flag in the insights block. Claude reads off who's
stable, who's getting flipped, and whether Amazon is in the mix. ~40
tokens cold.

> For ASIN B0CJT5D35W, what's the calibrated monthly sales estimate
> and how confident is the model in it?

`get_product_details` on a single ASIN. The `demand` block leads with a
`mode` field that tells you which estimation path fired: `read` (a
computed range with low / likely / high plus a confidence label of
`high`/`medium`/`low`), `badge` (Amazon's own "X+ bought past month"
figure, taken as ground truth), or `no-read` (signal too weak to
estimate, with a reason). This is a range estimate, not a forecast: the
confidence and mode matter as much as the number. ~8 tokens.

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
overlaid on a single chart at 800×400. 1 token. Claude Desktop's
regular chat renders the image inline. (Cowork, the agent-mode
surface, strips image content blocks before they reach the model, so
you'll get the text annotation but no picture there.)

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

Two-step. First call: `execute_keepa_finder` filters-only to learn
the match count. Second call: same filters plus `includeStats=true`,
which returns the `searchInsights` summary: average Buy Box price,
median seller count, the share of listings where Amazon is the Buy
Box winner, brand fragmentation across the match set, FBA share,
average rating, average review count. The stats call costs `30 + ⌈
totalResults / 1M⌉` tokens; refine first if the count is very large.

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
  minute, so long runs will queue as background jobs when the bucket
  runs low. That's normal at 1 TPM, not a fault: Claude polls the queue
  automatically with `check_job_status`, and it drains as tokens refill.
  Run `check_token_balance` any time to see what you have.
- **Background jobs run on your machine, not the cloud.** A queued
  product-details batch or code resolution drains as tokens refill, but
  only while a Claude app stays open. Quit Claude (or let the machine
  sleep) and the job pauses; relaunch and it resumes where it left off,
  nothing lost. Kick off the big one, leave Claude running, come back to
  results.
- **The 24-hour cache is real.** Re-asking the same question later the
  same day is usually free, and the cache is shared across chats and
  both Claude apps, so a fresh conversation re-reads it for nothing.
- **Stored results stay warm.** A finder result set or a code-resolution
  table is retrievable by id for 24 hours, so you can page back into it
  in a later message with `get_finder_result` / `get_codes_result`
  without re-running (and re-paying for) the original query.
- **Claude won't auto-chain expensive steps.** A finder result won't
  silently roll into a large product-details batch. Claude waits for you
  to ask. Compound asks in a single turn ("find X and deep-dive the top
  10") work as expected.
