# Troubleshooting

First-install behavior, log locations, and the handful of things most
likely to go sideways on your first run of Agellic Lite v1.5.0.

## 1. First-install behavior: what to expect

A brand-new install on a fresh machine boots the server in
**configuration-pending mode**. This is normal and expected. Here's what
you should see on each host:

### Claude Desktop

After you drag `agellic-lite.mcpb` into Settings → Extensions:

1. The credential form appears asking for your **Keepa API key** and,
   optionally, a **tokens-per-minute** value. Leave the TPM field blank
   and Agellic Lite auto-detects your key's real rate on first use (base
   Keepa is 1 TPM). There is no licence field: Agellic Lite is free.
2. The extension shows as **connected**, no red banner, no error icon.
3. Exactly **one tool** is visible in the tool list: `_configure_agellic`.
   Ask Claude "what tools do you have?" and it will surface this
   placeholder along with a description of what to enter and where.
4. Once you save the credential form, Claude Desktop auto-restarts the
   extension. After ~2-5 seconds the placeholder is replaced with the
   full **8-tool** set and you're ready to use Agellic Lite.

You should **not** see any error banner on a brand-new install. If you
see an error instead of the `_configure_agellic` placeholder, the
configuration-pending detection didn't fire. See [error #1](#1-brand-new-install-shows-an-error-instead-of-the-config-placeholder)
below.

### Claude Code

After you `unzip agellic-lite.zip` and run `node install.mjs`:

1. The installer prompts for your **Keepa API key** and, optionally, a
   **tokens-per-minute** value interactively.
2. You can skip the prompts by passing flags:
   ```
   node install.mjs --keepa-key <key>
   ```
   Add `--tpm <int>` only if you want to pin the rate; otherwise the
   boot probe auto-detects it.
3. The installer probes the server with your credentials before writing
   the MCP config entry, so if the Keepa key is wrong you'll see the
   error immediately, not on first tool call.
4. There is no placeholder mode on Claude Code: either the install
   completes with all 8 tools available, or it fails with a specific
   error you can act on.

## 2. Log locations

There are two logs worth knowing about. Claude Desktop writes its own
per-server log, and Agellic Lite writes its own server-side log too.

### Claude Desktop's per-server log

Claude Desktop writes a per-server log under its logs directory. The filename
starts with `mcp-server-` and is derived from the Agellic Lite extension name,
so match by that prefix:

- **macOS:** `~/Library/Logs/Claude/`
- **Windows:** `%APPDATA%\Claude\Logs\`

This captures stderr from the server process plus Claude Desktop's own
view of the MCP handshake. Useful for connection-level issues.

### Agellic Lite's own log (per-day rotation)

- **macOS / Linux:** `~/.agellic-lite/logs/agellic-<YYYY-MM-DD>.log`
- **Windows:** `%USERPROFILE%\.agellic-lite\logs\agellic-<YYYY-MM-DD>.log`

This is the server-side log with full request/response tracing, token
bucket accounting, and credential-resolution diagnostics. The server
prints the active log path to stderr on boot, so the first few lines of
the Claude Desktop log will tell you exactly where to look. If you set
`AGELLIC_LITE_DATA_DIR`, the logs move under that directory instead.

**Both logs redact Keepa API keys.** You can paste log excerpts into a
support email without leaking your key.

### Log volume controls

Two environment variables raise the log ceiling:

- `AGELLIC_LOG_LEVEL`: one of `error | warn | info | debug`
  (default `info`).
- `AGELLIC_LOG_DEBUG`: comma-separated list of module names to
  elevate to debug, e.g. `tool-call,token-bucket`. Useful when you
  want debug detail from just one subsystem without the firehose.

On Claude Desktop, set these in the credential form's "Advanced" pane
(if exposed) or in your system environment before launching CD. On
Claude Code, set them in the `agellic-lite` MCP server `env` block in
your `~/.claude.json` or the equivalent CC config file.

## 3. Common errors

### 1. Brand-new install shows an error instead of the config placeholder

**Symptom:** You just installed the `.mcpb` for the first time, opened
the credential form, and Claude Desktop is showing an error instead of
the `_configure_agellic` placeholder tool, before you've entered
anything.

**Cause:** The configuration-pending detection didn't fire. The server
handles a Claude Desktop quirk where unsubstituted `${user_config.X}`
placeholders are passed as **literal text** (not empty string) when the
credential form is blank. This is defended against in current code.

**Remedy:** Open Claude Desktop's per-server log in
`~/Library/Logs/Claude/` (macOS) or `%APPDATA%\Claude\Logs\` (Windows), the
file whose name starts `mcp-server-` and references Agellic Lite, and search
for `first-install-detected`. If that line is missing, the
defense didn't engage on your machine. File an issue with the log
excerpt attached.

### 2. Keepa key is mangled when pasted into Claude Desktop

**Symptom:** Your Keepa key works on the Keepa dashboard but fails here,
or the server reports the key as invalid right after you paste it.

**Cause:** Claude Desktop's credential form has two paste-time
mutations that affect long string values:

- **Fields marked `sensitive: true`** (the Keepa key), values get
  soft-wrapped at roughly 76 columns, with `\n` characters inserted at
  the wrap points.
- **Fields marked `sensitive: false`**, newlines are stripped from
  pasted multi-line content.

Both are upstream Claude Desktop bugs, not Agellic bugs. The server
defends against both: the Keepa key is whitespace-stripped at the
env-read boundary before any validation runs.

**Remedy:** Most paste-time mangling is silently corrected by the
server. If your key still fails after a clean re-paste, confirm it's
active on the Keepa dashboard, then capture the log and file an issue.

### 3. Wrong-edition data directory: Lite refuses a full-server data dir

**Symptom:** The server fails to boot, and the log says the configured
data directory belongs to the other Agellic edition (the full Agellic
server), or the full server refuses a directory that Agellic Lite
stamped.

**Cause:** Agellic Lite and the full Agellic server each own a separate
data directory (`~/.agellic-lite` for Lite, `~/.agellic-mcp` for the
full server) and each refuses to open the other's, so the two editions
can coexist on one machine without cross-contaminating credentials,
caches, or the token bucket. You'll hit this if `AGELLIC_LITE_DATA_DIR`
points at a directory a full Agellic install already owns (or vice
versa).

**Remedy:** Leave `AGELLIC_LITE_DATA_DIR` unset so Agellic Lite uses its
default `~/.agellic-lite`, or point it at a fresh directory of its own.
Never share one data directory between the two editions.

### 4. Node version error on Claude Code

**Symptom:** `node install.mjs` or the server itself exits with a
message about an unsupported Node.js version.

**Cause:** The Claude Code build runs on your system Node, and Agellic
Lite needs a recent release: **Node 22.22.2+ or 24.15.0+**. (Claude
Desktop ships its own Node, so this only affects Claude Code installs.)

**Remedy:** Upgrade Node to a supported version, then re-run
`node install.mjs`. Check your version with `node --version`.

### 5. The token bucket is empty: your work is queued

**Symptom:** A tool call returns a `wait <N> minutes` message, or you
see `token-bucket: exhausted` in the Agellic Lite log, or a costly call
comes back with a `jobId` instead of an immediate result.

**Cause:** You've spent your available Keepa tokens for the moment. On
base Keepa (1 TPM) the bucket holds 60 tokens and refills 1 per minute,
so any batch larger than that has to wait for refills. This is normal,
not a failure.

**Remedy:** Nothing to do. Larger lookups queue as a background job:
poll with `check_job_status` and the result lands once the bucket has
refilled enough to cover it. You don't need to re-issue the request. If
your Keepa subscription actually allows a higher refill rate than 1 TPM,
set `AGELLIC_LITE_TOKENS_PER_MINUTE` to that value (or let the boot
probe auto-detect it) and the queue drains faster.

### 6. Claude Desktop: the 8 tools don't appear after install

**Symptom:** You installed the `.mcpb`, the extension shows as
connected, but the tool list shows fewer than 8 tools (or only
`_configure_agellic` persists after you've saved credentials).

**Cause:** Claude Desktop only re-reads its extension's tool list on
cold start. A window close (Cmd+W on macOS, or the X button on Windows)
doesn't count: the app process is still running.

**Remedy:** Fully quit Claude Desktop (**Cmd+Q on macOS**, or right-click
the tray icon → Quit on Windows) and reopen. The full 8-tool set will
appear.

### 7. Cowork shows charts as text-only

**Symptom:** You asked for a product price chart via Cowork (Claude
Desktop's agent-mode surface) and got a text summary plus the Keepa URL
but no rendered image. The same tool call from regular Claude Desktop
chat or Claude Code renders the chart fine.

**Cause:** Cowork runs the agent inside a sandboxed VM that (a) doesn't
paint inline `type: 'image'` content blocks for the user and (b) blocks
reads of files written outside the session's allowed directories, so a
host-saved chart PNG can't be reached either. The chart is generated
successfully and the model still receives the image for analysis; only
the inline display to the user is unavailable on this surface.

**Remedy:** None on the Agellic side. This is a Cowork sandbox
constraint, not an Agellic bug. In Cowork, use the data readout plus the
Keepa product URL the tool returns (pair the chart request with
`get_product_details` for a fuller picture). When you want the rendered
chart, use regular Claude Desktop chat or Claude Code.

### 8. A background job is stuck at `pending` or `running`

**Symptom:** A tool call returned a `jobId` (a large product-details
batch, a big code resolution, or a finder), but `check_job_status` keeps
reporting `pending` or `running` and the result never lands.

**Cause:** Two normal, non-error reasons, usually one of:

- **Waiting for tokens.** A big job needs the Keepa bucket to refill
  enough capacity before it can run. On base Keepa (1 TPM) a 100-ASIN
  details batch (~800 tokens) takes many refill cycles. The job stays
  `pending` until the bucket can cover it. `check_job_status` reports
  the queue position, the funding gate, and an ETA so you can see it's
  progressing.
- **The host isn't running.** The job runner lives inside the MCP server
  process, which only runs while Claude Desktop or Claude Code is open.
  Quit Claude or let the machine sleep/shut down and the queue stops
  making progress: there is no cloud worker.

**Remedy:** Leave a Claude app open and the machine awake; the job drains
on its own as tokens refill. If you did quit, nothing is lost: the job's
lease is reclaimed on the next launch and it resumes where it left off.
Poll with `check_job_status`. To abandon a job you no longer want, ask
Claude to cancel it via `check_job_status`, which exposes a cancel
action.
