# Changelog

All notable changes to Agellic Lite are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

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
