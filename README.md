# Agellic Lite: free Amazon product intelligence for AI assistants

**Ask in plain English, get a decision, not a spreadsheet.** Calibrated
demand estimates, Buy Box and stock signals, price and rank history, and
natural-language product discovery, powered by your own
[Keepa](https://keepa.com/) key. Free, and no Agellic account needed.

Agellic Lite is a local [Model Context Protocol](https://modelcontextprotocol.io)
(MCP) server, so it runs in any MCP-compatible assistant. Today it ships
with one-click installers for **Claude Desktop** and **Claude Code**, with
more hosts as the MCP ecosystem grows.

> Agellic Lite is the free edition. Bring your own Keepa API key and you are
> ready to go: no license token, no signup, no Agellic account. If you want
> bulk screening and cross-marketplace arbitrage, those live in the full
> Agellic server (invite-only).

## Why Agellic Lite

**Answers, not spreadsheets.** It reads Keepa's raw history and hands the
model the *conclusions* (Buy Box health, stock depth, rank trends,
out-of-stock patterns, demand stability, seasonality) so you ask in plain
English and get a judgment, not a CSV to squint at.

**Demand Read.** A calibrated demand-estimation model: a real read on how a
product moves, not just "BSR #14,000." The full methodology is written up in
[COMPUTED-INSIGHTS.md](./COMPUTED-INSIGHTS.md).

**Built for a base Keepa plan.** Base Keepa refills 1 token per minute, and a
single deep product read costs several tokens. Agellic Lite is built around
that limit instead of fighting it: costs are checked before spending, failed
calls are refunded, cached reads are free, and anything too big to run right
now **queues and drains itself** as your tokens refill. See
[TOKENS-AND-QUEUE.md](./TOKENS-AND-QUEUE.md).

**Background jobs that drain themselves, locally.** Too big to run right now?
It queues and works the backlog automatically as Keepa tokens refill, no
babysitting. It runs on **your machine, not the cloud: leave the app open and
the machine awake for jobs to progress**, and the queue is durable, so
quitting pauses it and relaunching resumes right where it left off.

**Remembers across chats.** Fetched products and result sets are cached on
your machine and shared across every chat and every connected app, so you
never pay Keepa twice for the same lookup, and you can reopen a finder run in
a fresh conversation for free.

**Natural-language product discovery.** Describe what you want ("car
accessories under $30, rank under 50k") and it builds the Keepa Product Finder
query, returns the match count, then lets you page through and rank the hits.
No filter-form wrangling.

**Supplier-manifest matching.** Paste a list of UPC, EAN, or GTIN codes and it
resolves them to candidate ASINs in bulk, with a cached table you page
through. Turning a supplier sheet into ASINs by hand is tedious; this is one
question.

**Built for the model.** Compact, structured output that fits the context
window (cheaper, faster, sharper answers), plus price and BSR charts the model
can actually see and read.

**One paste, set up once.** Runs in Claude Desktop and Claude Code today.
Configure once per machine (shared credentials), with category and demand
calibration data bundled in, no extra downloads.

## Latest release

[**Download from /releases/latest**](https://github.com/Agellic-Commerce/agellic-lite/releases/latest)

## Which file do I download?

| Host                             | File                | Recipe                                 |
| -------------------------------- | ------------------- | -------------------------------------- |
| Claude Desktop (macOS / Windows) | `agellic-lite.mcpb` | Drag into CD, Settings, Extensions     |
| Claude Code (macOS / Linux)      | `agellic-lite.zip`  | `unzip` + `node install.mjs`           |
| Claude Code (Windows)            | `agellic-lite.zip`  | `Expand-Archive` + `node install.mjs`  |

Both files are **byte-identical**; just two filenames so each host's unzip
tool recognises the extension.

## Prerequisites

- **Keepa API key**: get one at [keepa.com/#!api](https://keepa.com/#!api). A
  base Keepa subscription is enough. This is the only credential Agellic Lite
  needs.
- **Node.js 22.22.2+ or 24.15.0+** (Claude Code only; Claude Desktop ships its
  own Node runtime).

No Agellic license token. No Agellic account. You only ever pay Keepa, on your
own Keepa subscription.

## Documentation

- [**Install guide**](./INSTALL.md): requirements, step-by-step for Claude
  Desktop and Claude Code, plus upgrade and uninstall.
- [**Tool reference**](./TOOLS.md): the 8 tools, what each one does, and what
  it costs in Keepa tokens.
- [**Tokens and the queue**](./TOKENS-AND-QUEUE.md): how the token budget, the
  cache, and the background queue work together on a base Keepa plan.
- [**Computed insights**](./COMPUTED-INSIGHTS.md): the algorithms behind every
  number `get_product_details` returns.
- [**Usage examples**](./USAGE.md): example prompts to try first.
- [**Troubleshooting**](./TROUBLESHOOTING.md): first-install behavior, log
  locations, common errors.
- [**FAQ**](./FAQ.md): Keepa token economics, privacy, compatibility.
- [**Changelog**](./CHANGELOG.md): release notes.
- [**License**](./LICENSE.md): free-of-charge terms, AS-IS, no warranty.

## Release status

This is the **v1.5.0 release** of Agellic Lite, a free edition distributed
openly. Today's artifacts are unsigned builds; the trust stack (Sigstore and
cosign supply-chain attestation, signed checksums, and an SBOM) is on the
roadmap. Until then, install only from `Agellic-Commerce/agellic-lite` and
check file sizes against the release page before running.
