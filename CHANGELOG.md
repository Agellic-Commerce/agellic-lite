# Changelog

All notable changes to Agellic Lite are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [2.0.1] - 2026-08-12

Data-honesty patch release: fixes shared with our web pipeline so a
"no data" marker can never read as a real number, plus removal of a
documented field that never actually appeared. No new tools and no
cache format change.

### Fixed

- **Sales-rank drop counts no longer show Keepa's "no data" marker as a
  number.** When Keepa has no drop data it reports -1; that could surface
  as a literal negative drops count. It now reads as null (no data).
- **Offer price history excludes both of Keepa's no-value markers.** An
  "absent value" (-2) tick could previously enter price history as a
  negative price and skew the price-compression read toward a false
  "compressed" signal. Neither marker is ever treated as a price now.
- **Price-position now shows its measurement basis.** The output names
  which price lane (buy-box vs lowest-new) and which shipping convention
  the percentile and z-score were measured on. The fields were always
  computed but were dropped from the rendered output.

### Removed

- **"Rank volatility" no longer appears in docs or tool descriptions.**
  It was documented but never actually emitted (dead code since it
  shipped), so nothing you relied on changes. Buy Box volatility, a
  different and live insight, is unchanged.

## [2.0.0] - 2026-08-11

Every Keepa-calling tool call is now a durable work order. Work that
used to wait invisibly (or fail) when tokens ran short now accepts,
survives restarts, runs itself as tokens refill, and reports honest
progress and an honest wait. Nothing expensive starts without your
consent. Agellic Lite stays free and license-free: your Keepa key is
still the only credential.

### Added

- **`confirm_work_order` joins the Lite roster.** Work quoted above
  roughly one hour of token refill does not start. The call returns the
  order id, a cost-only quote, a queue-aware ETA, and a one-time
  confirm token; nothing is charged until you agree and the assistant
  passes the token back. Confirming queues the order (authorised is not
  started, and the status view tells you which is true).
- **Mid-run row access.** `check_job_status` action `fetch` pages
  through rows an order has already settled, mid-run or after it
  finishes.

### Changed

- **Every call returns one of three answers.** The result right away
  when the token balance covers the quote; a background acceptance when
  it does not (the order then runs itself as tokens refill); or a quote
  plus confirm token when the work is big enough to need consent. All
  three name the order id and the result-set handle your next call can
  reuse.
- **Cost and time are two separate claims.** Acceptances state the flat
  token cost on one line and the wait on another: a bound ("done within
  ~X of token refill"), recomputed on every poll so it counts down as
  the queue drains. The bound covers the work as currently quoted,
  assumes no other work on your Keepa key, and its wall-clock figure
  assumes a session open about 8 h/day (an always-open session finishes
  about 3x sooner).
- **`check_job_status` is the hub.** Modes list, status, cancel, and
  fetch. Status separates `authorised:` (you agreed) from `started:`
  (a Keepa request has actually been dispatched), and withholds the
  ETA for orders that will not finish on refill alone. An order
  awaiting consent shows its ETA conditionally, prefixed "If you
  confirm now:".
- **Charges settle at Keepa's own figure.** Quotes are worst-case
  ceilings, and most orders settle under them. Mid-run, the charged
  total can include tokens reserved for requests still in flight;
  those release when the request settles.
- **Cancel keeps what you paid for.** Cancelling a not-yet-started
  order stops it outright at zero charge. Cancelling a running order
  stops it at its next dispatch boundary, and everything settled so
  far stays readable.

### Removed

- **The rate-limit-wait job queue.** Nothing queues invisibly anymore.
  Anything big enough to wait is a work order you can see, poll, and
  cancel, and it survives a session restart.

### Upgrade notes

- A Claude Code CLI install clears the local product cache on upgrade,
  so previously cached products refetch once at their normal token
  price. Claude Desktop (.mcpb) upgrades keep the cache.
- Background work only advances while an Agellic Lite session is open.

## [1.8.0] - 2026-08-01

Buy Box prices now include shipping. Every Buy Box number this server
reports is the LANDED price (item + shipping), which is what a buyer
actually pays and what Amazon charges its referral fee on. Expect a
one-time value shift on shipped items: yesterday's $10.99 may read
$15.98 today with no offer change at all. The item price did not move,
the label got honest. Items with free shipping are unchanged.

### Changed

- **Buy Box prices are landed (item + shipping) on every surface.**
  This is Keepa's native `csv[18]` series, which carries shipping across
  the full 1095 days of history, so this lane has no cutover date and no
  mixed-convention window. The change reaches `pricing.buyBox.*`
  (current, the 30/90/180/365d averages, and the 365d min/max),
  `competition.buyBox.priceCents`, the Buy Box lane of the sell-price
  read, the price position insight, and the finder's `avgBuyBox` insight.
- **Re-baseline any saved Buy Box thresholds.** A price filter, ROI
  target, or alert level tuned against pre-1.8.0 item-only Buy Box
  numbers is now being compared against a larger number on every shipped
  item. This is the one thing worth checking after you upgrade.
- **Offers sort by landed price.** The offer list in
  `get_product_details` now orders by what you would actually pay, so
  the offer at the top is the cheapest to receive rather than the one
  with the lowest sticker and a shipping charge hidden behind it.
- **Finder price filters were always landed, and now say so.**
  `current_BUY_BOX_SHIPPING`, `delta30/90_BUY_BOX_SHIPPING`, and
  `buyBoxStandardDeviation30/90/365` are Keepa fields that have always
  included shipping (the `_SHIPPING` suffix is the tell). Only the
  wording changed here, no filter behavior did.

### Added

- **`pricing.buyBox.shippingBasis: 'landed'`** marks the new contract.
  Its absence marks a stale pre-landed payload, so anything reading
  these fields programmatically can tell the two apart without guessing.
- **Reconciliation components.** `pricing.buyBox.itemCents` and
  `pricing.buyBox.shippingCents` show exactly how a landed price splits.
  They appear only when shipping is greater than zero, so free-shipping
  items carry neither.
- **Basis disclosure on the read surfaces.** `pricing.sellPrice` and the
  price position insight now declare a `shippingBasis` of `landed`,
  `item-only`, or `mixed-window`. The last one means the window straddles
  the date Keepa started including shipping in its lowest-new series, so
  the prices inside it mix both conventions and are not comparable to
  each other.
- **`pricing.sellPrice.marketState: 'suppressed'`** on listings whose
  Buy Box is suppressed. A normal market has no state worth reporting,
  so the field is present only when there is something to disclose.

### Fixed

- **The lowest-new shipping-inclusion date was a week early.** Keepa
  began including shipping in its lowest-new series on 2026-02-23, not
  2026-02-16. The earlier date comes from a comment bug in Keepa's own
  Java client. Reads whose window crosses that boundary now caveat
  against the right instant.
- **Honest wording when a listing only ever held two prices.** The
  time-at-price hero used to print the entire coverage window as though
  a single price had held for all of it. It now reports the dominant
  price with its real dwell time, or the range across both levels.
- **Chart captions scope the landed claim to the Buy Box curve.** The
  new-offer curve is item-only before 2026-02-23, and the caption now
  says so instead of implying every curve on the chart is landed.

## [1.7.1] - 2026-07-31

This release puts the price/BSR chart in front of you on two more
surfaces: the ChatGPT desktop app (as an inline card, opt-in via the
installer) and Cowork, which renders charts inline for the first time.

### Added

- **Inline chart cards in the ChatGPT desktop app.** `get_product_chart`
  now renders as an MCP Apps card in the conversation when Codex's
  `enable_mcp_apps` feature flag is on and you are signed in to ChatGPT.
  Without the flag the chart still reaches the model; ChatGPT just shows
  it inside the tool-call expander.
- **The codex installer offers that flag, never flips it silently.**
  Interactive `--host codex` installs and upgrades ask (default No);
  non-interactive runs only act on the new `--enable-mcp-apps` flag. The
  enable goes through `codex features enable` (Codex's own config
  editor; your `config.toml` is never hand-edited), and a failure there
  never fails the install. An explicit `enable_mcp_apps = false` or
  `apps = false` in your config is always respected, and uninstall
  leaves the flag alone. Expect a harmless "under-development features"
  notice at codex startup once enabled. See
  [INSTALL.md](./INSTALL.md#inline-chart-cards-in-chatgpt-desktop-optional).

### Fixed

- **Cowork renders charts inline.** Charts now arrive in Cowork (Claude
  Desktop's agent-mode surface) as the same MCP Apps view regular chat
  uses; earlier releases could only deliver a text readout there. Chart
  display in Claude Desktop chat also survives a recent Claude Desktop
  update that changed how tool results reach the chart view. If Cowork
  still shows text-only charts after upgrading, quit Claude Desktop
  fully (Cmd-Q) and reopen: Cowork keeps the previous server process
  alive until a full quit.

### Changed

- **Chart summaries no longer include a Keepa browser URL line.** The
  inline render (or the tool-call expander on surfaces without one) is
  the chart channel; the URL line was a leftover fallback.

## [1.7.0] - 2026-07-29

This release answers a question Agellic Lite could not answer before, "what
can I actually sell this for?", and rebuilds the seasonality detector so
that a confirmed season means the peak genuinely recurred across years.

### Added

- **Sell-price read in `get_product_details`.** Every product now carries
  `pricing.sellPrice`: observed sale-price bands (`moveFastCents` /
  `marketCents` / `stretchCents`, the 25th / 50th / 75th percentiles of the
  prices in force at inferred sale moments), the current price's position
  inside them, sales skew, 30-day drift, and honest caveats. Three methods
  ladder from transaction-quality on down: Buy Box prices at sale events,
  the lowest-new floor at sale events when the Buy Box is suppressed, and
  duration-weighted time-at-price when sale events are too thin; below
  that, plain window averages. The bands describe the observed market,
  never a pricing recommendation. See
  [COMPUTED-INSIGHTS.md section 3](./COMPUTED-INSIGHTS.md#3-sell-price-read).
- **`insights.recentDemandDeviation`.** The trailing-year average rank
  divided by the trailing-30-day average: a one-number read on whether
  demand is currently running above or below the product's own baseline.

### Changed

- **Seasonality detection rebuilt around recurrence.** A seasonal peak is
  now `confirmed` only when the same peak window recurred across at least
  two separate years of history AND beat a statistical null test that
  rejects slow rank drift masquerading as seasonality. A single observed
  season reports as `candidate` with explicit "do not act on this alone"
  framing. Detection itself moved to a scale-free cluster split of the
  weekly rank profile. Deliberately conservative; verified against two
  blind holdout sets with zero false confirmations. See the rewritten
  [COMPUTED-INSIGHTS.md section 2](./COMPUTED-INSIGHTS.md#2-seasonality).
- **Three years of history for the deep read.** `get_product_details` now
  fetches 1095 days of history (was 365), giving seasonality confirmation
  up to three full cycles to work with; per-ASIN token cost is unchanged.
  Seasonality computed from shorter windows (under 730 days) is explicitly
  capped at `candidate` and its summary says why.
- **Honest calendar labels.** A peak is labeled Q4 / Summer / Halloween
  etc. only when it genuinely matches the calendar template: the overlap
  bar tightened, plus a new containment path for a narrow peak sitting
  inside a wide season window. Otherwise the label is `Other` and the peak
  week range carries the identity.
- **The product cache resets on this upgrade** (new cache format for the
  three-year window): previously cached ASINs refetch at the normal token
  cost on first read. Your Keepa key, token state, and logs are untouched.

## [1.6.0] - 2026-07-20

Agellic Lite now installs into Codex CLI and the ChatGPT desktop app
alongside Claude Desktop and Claude Code: one command registers both Codex
surfaces, your Keepa key never enters Codex configuration, and every host on
the machine shares one credential cache, one data dir, and one job queue.

### Added

- **Codex CLI + ChatGPT desktop as installable hosts.** `node install.mjs
  --host codex` installs a Codex-owned copy of the server, probes it with a
  real MCP round-trip before touching any configuration, and registers it
  through the `codex` CLI. Codex CLI and ChatGPT desktop share MCP
  configuration, so the one command covers both. See
  [INSTALL.md](./INSTALL.md) for the walkthrough.
- **Credential-free Codex configuration.** Your Keepa key is written only to
  the per-machine credential cache; the `agellic-lite` Codex entry itself
  carries no secrets, and the server reads the cache at boot.
- **Promptless second-host install.** If any host already configured Agellic
  Lite on this machine, `node install.mjs --host codex --non-interactive`
  completes with no prompts, entirely off the shared cache.
- **One-command upgrades across hosts.** `node install.mjs --host all
  --upgrade` detects which scripted hosts are installed (Claude Code, Codex)
  and upgrades each in turn with a per-host report.
- **Per-host uninstall with a safe purge order.** Every uninstall level
  (config only, config + binary, full purge) now works per host, and the
  full purge refuses while another host is still installed, naming it, so
  you can't corrupt a live host's shared data. INSTALL.md documents the
  order that always works without `--force`.
- **A graceful path when the `codex` CLI is missing.** The installer still
  installs and probes the server and writes the credential cache, then
  prints two remedies: install the CLI and re-run (always safe, the
  registration is idempotent), or add the server manually in ChatGPT
  desktop settings.

## [1.5.0] - 2026-07-04

First public release of Agellic Lite, the free edition of the Agellic MCP
server. Bring your own Keepa API key and you're ready to go. It shares its
version line with the full Agellic server.

### Added

- **Eight tools for Amazon product intelligence.** `get_product_details`
  (calibrated demand, seasonality, and insights), `get_product_chart` (inline
  price and BSR chart), `execute_keepa_finder` and `get_finder_result`
  (natural-language product discovery you can page through and rank),
  `resolve_codes` and `get_codes_result` (bulk UPC / EAN / GTIN to candidate
  ASINs), `check_token_balance` (cost preview without spending), and
  `check_job_status` (the background queue).
- **Built for a base Keepa plan.** Agellic Lite floors its rate policy at 1
  token per minute so a base Keepa key stays usable. Costs are checked before
  spending, cached reads are free, failed calls are refunded, and anything too
  big to run right now queues and drains itself as tokens refill. See
  [TOKENS-AND-QUEUE.md](./TOKENS-AND-QUEUE.md).
- **Accurate first-call pricing.** On the first costly call after startup,
  Agellic Lite runs a free check of your real Keepa balance and rate and aligns
  its local budget to it, so cost estimates match what Keepa will charge and
  you rarely need to set a rate by hand.
- **A visible, cancellable queue.** `check_job_status` shows a pending job's
  position in line, the balance it is waiting to reach, and an ETA from your
  refill rate, and it can cancel a job that has not started yet. The queue is
  durable across restarts and collapses duplicate requests onto the job already
  in flight.
- **A local, shared cache.** Fetched products and result sets are cached on
  your machine and shared across every chat and both hosts, so you never pay
  Keepa twice for the same lookup.
- **One-click install for Claude Desktop and Claude Code.** Drag the
  `agellic-lite.mcpb` into Claude Desktop, or unzip `agellic-lite.zip` and run
  `node install.mjs` for Claude Code. Category and demand calibration data are
  bundled in; no extra downloads.
- **Coexists with the full Agellic server.** Its own MCP entry
  (`agellic-lite`), bin directory (`Agellic-Lite`), and data dir
  (`~/.agellic-lite`), fully separate from a full install. Each edition refuses
  to run against the other's data dir.
