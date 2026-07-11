# Changelog

All notable changes to Agellic Lite are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

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
