# FAQ

Answers to questions people actually ask about Agellic Lite.

## Keepa tokens

### What's a Keepa token?

A Keepa token is a unit of API quota issued by Keepa, the data vendor
Agellic Lite sits on top of. Every tool call consumes some number of
tokens against the Keepa API key you supply at install. Agellic Lite
itself is free: the only thing you pay for is your own Keepa
subscription. Per-tool costs are listed in [TOOLS.md](./TOOLS.md).

### Why do tools have different costs?

Discovery is cheap; per-ASIN enrichment costs more, scaling with the
depth of data each tool needs:

| Tool | Cost | What it pays for |
|---|---|---|
| `execute_keepa_finder` | 10 + 1 per 100 ASINs returned | discovery against Keepa's pre-aggregated finder index |
| `resolve_codes` | ~1 token / returned candidate (≈1 per code typical) | identity-tier UPC/EAN/GTIN/ISBN to candidate-ASIN lookup |
| `get_product_details` | ~8 tokens / ASIN (9 reserved, unused graph token refunded) | full product record (offers, history, dimensions, calibrated demand) |
| `get_product_chart` | 1 token / chart | rendered PNG from Keepa |

`check_token_balance`, `check_job_status`, and the `get_*_result`
readers are local: they cost nothing.

### What happens when I run out of tokens?

Nothing is lost. The server enqueues a background job and returns a
`jobId`. Poll it with `check_job_status`. The token bucket refills at
`AGELLIC_LITE_TOKENS_PER_MINUTE` per minute (base Keepa is 1 TPM, which
Agellic Lite auto-detects on first use). Once the bucket has enough
capacity to cover the job, it runs. Until then the job stays `pending`:
no work lost, no retry needed on your end.

At 1 TPM the bucket holds 60 tokens and refills 1 per minute, so most
real batches are larger than a single bucket. That is expected. The
queue is the normal way Agellic Lite works, not a fallback: it drains
your backlog automatically as Keepa tokens refill. Kick off the big
lookup, leave Claude running, and come back to results.

### Do tokens carry over month-to-month?

That's a Keepa subscription question, not an Agellic one. Check your
Keepa account dashboard at [keepa.com/#!api](https://keepa.com/#!api).

### Where do I check my balance?

Ask Claude to run `check_token_balance`. It's a free local read:
no Keepa call, no cost.

## Caching & background jobs

### If I look up the same product twice, do I pay twice?

No. Every paid fetch is cached on your machine (products for 24 hours)
and the cache is shared across every chat and both Claude apps.
Re-asking about the same ASIN, even in a brand-new conversation or the
other host, reads from cache at **zero Keepa tokens**. You can also
reopen a stored finder or code-resolution result by id long after the
original chat scrolled away: see `get_finder_result` /
`get_codes_result` in [TOOLS.md](./TOOLS.md).

### Do background jobs keep running if I close Claude?

Only while a Claude app is open. The job runner lives inside the MCP
server process, which Claude Desktop and Claude Code start and stop with
the app: there's no cloud worker. Jobs are durable, though: quitting
Claude (or restarting your machine) **pauses** the queue, and the next
time you launch Claude the job resumes from where it left off: nothing
is lost. So for a big product-details batch or a large code-resolution
run, kick it off and leave Claude running; it drains as your Keepa
tokens refill.

## About Agellic Lite

### Is Agellic Lite really free?

Yes. Agellic Lite is a free, public edition of the Agellic MCP server.
You bring your own Keepa API key and pay only Keepa, on whatever
Keepa subscription you already have. The base Keepa tier (1 TPM) is
enough to get started; larger token allowances just drain the queue
faster.

### Is this finished software?

Agellic Lite is early software, shipped **as-is**. It's stable and used
daily, but treat it as a decision-support tool, not an oracle: verify
outputs against Keepa and your own judgement before committing money.
There's no automatic updater yet, so pull new releases from the repo
when you want the latest. Questions or issues: `support@agellic.com`.

## Privacy

### What does the MCP send to Keepa?

Only the call parameters you'd send if you used Keepa's API directly:
the ASIN list, the marketplace domain code, and the filter parameters
you specified. No conversation transcript, no Claude context, no PII.
Each call goes from your machine directly to `api.keepa.com` over
HTTPS.

### Are my queries logged?

Yes, locally, at `~/.agellic-lite/logs/agellic-<date>.log` (rotated
daily, 14-day retention by default). Keepa API keys are redacted before
they touch the log. Logs stay on your machine; Agellic does not collect
telemetry and the server makes no outbound calls except to Keepa.

### Does Agellic see my Claude conversations?

No. The MCP boundary only exposes the tool-call arguments Claude
chooses to send (e.g. an ASIN list, a domain code). The surrounding
conversation, your prompts, and Claude's reasoning are never sent
to the Agellic Lite process.

## Compatibility

### Which clients does this work in?

- **Claude Desktop** (macOS, Windows): install via the `agellic-lite.mcpb`
  extension. Drag the file into Settings → Extensions.
- **Claude Code** (macOS, Linux, Windows): install via
  `node install.mjs` after unzipping `agellic-lite.zip`.
- **Codex CLI + ChatGPT desktop** (macOS, Linux, Windows): install via
  `node install.mjs --host codex` after unzipping the same archive.
  One command registers both; they share MCP configuration.

Agellic Lite is a standard local MCP server, so other MCP-capable hosts
can run it too; the hosts above are the supported install paths.

Linux desktop is not supported for Claude Desktop because CD itself
doesn't ship a Linux build. If/when that changes, the existing `.mcpb`
will work there too.

### Cowork limitations?

In Cowork (Claude Desktop's agent-mode surface), `get_product_chart`
can't display the chart image: the sandboxed VM doesn't paint inline
image content blocks and blocks reads of host-saved files. The model
still receives the chart for analysis, so you get an accurate readout
plus the Keepa product URL, just not the inline picture. Every other
tool works normally. Regular Claude Desktop chat and Claude Code both
render the chart inline.

When you need the chart visual, use a regular Claude Desktop chat thread
or Claude Code.
