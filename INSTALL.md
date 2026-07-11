# Install guide

How to install, upgrade, and uninstall Agellic Lite (`agellic-lite`) in
Claude Desktop and Claude Code.

For things that can go wrong, see [TROUBLESHOOTING.md](./TROUBLESHOOTING.md).
For example prompts once installed, see [USAGE.md](./USAGE.md).

---

## Requirements

- **A Keepa API key**: get one at [keepa.com/#!api](https://keepa.com/#!api).
  A base Keepa subscription is enough. This is the only credential Agellic
  Lite needs.
- **Node.js 22.22.2+ (LTS) or 24.15.0+** (Claude Code only; Claude Desktop
  ships its own Node runtime). Node 20 reached end of life on 2026-04-30 and
  is no longer supported. If you're on older Node, install a current LTS from
  [nodejs.org/en/download](https://nodejs.org/en/download) (or via `nvm`,
  `fnm`, `volta`, etc.).
- **One of:**
  - **Claude Desktop** (macOS or Windows; Linux is not supported by CD
    itself).
  - **Claude Code** (any platform: macOS, Linux, Windows).

You can install into both hosts on the same machine. They share credentials
at `~/.agellic-lite/credentials-lite.json` (mode 0600), so after the first
host is configured, the second host's install can leave the Keepa field blank
and pick everything up from the cache. See [Where files live](#where-files-live).

Agellic Lite installs fully separately from the full Agellic server (its own
MCP entry, its own bin directory, its own data dir), so the two can coexist on
one machine without touching each other.

---

## Install in Claude Desktop

1. Download `agellic-lite.mcpb` from the
   [latest release](https://github.com/Agellic-Commerce/agellic-lite/releases/latest).
2. Open Claude Desktop, then **Settings**, then **Extensions**, and install
   the `.mcpb` one of two ways:
   - **Drag and drop** the `.mcpb` file into the drop target; or
   - If your build doesn't show a drop target, click **Advanced settings**,
     then **Install extension**, and select the downloaded `.mcpb`.
3. Paste your **Keepa API key** into the credential form. Leave **Tokens per
   minute** blank to auto-detect your Keepa plan's rate on first use (base
   Keepa is 1), or set it explicitly if you know your tier.
4. Claude Desktop installs the extension and auto-restarts it. After a few
   seconds the 8 Agellic Lite tools appear in the tool list.

The `.mcpb` file location doesn't matter; CD reads the file wherever you drag
from and copies the contents into its own extension directory at
`~/Library/Application Support/Claude/Claude Extensions/local.mcpb.agellic.agellic-lite/`
(macOS) or `%APPDATA%\Claude\Claude Extensions\local.mcpb.agellic.agellic-lite\`
(Windows). After install, you can delete or move the original `.mcpb` freely.

### First-install behavior (Claude Desktop)

On a brand-new install with no credential cache on the machine, the server
boots in **configuration-pending mode**:

- The extension shows as connected; one placeholder tool named
  `_configure_agellic` appears in the tool list. Asking Claude what tools it
  has will surface that one tool with a description telling you exactly what to
  do.
- Paste your Keepa key into the form and click **Save**. Claude Desktop
  restarts the extension automatically.
- After a few seconds, the placeholder is replaced with the full 8-tool set.

You should **not** see a red "could not find a valid license" banner: Agellic
Lite has no license to find. If a tool call reports a missing Keepa key, open
the extension settings and paste the key.

If you already configured Agellic Lite in Claude Code on this machine, leave
the Keepa field **blank** in the CD form: it will be filled from the
per-machine credential cache automatically.

---

## Install in Claude Code

### macOS / Linux

```bash
curl -fsSL -O https://github.com/Agellic-Commerce/agellic-lite/releases/latest/download/agellic-lite.zip
unzip agellic-lite.zip -d agellic-lite-install
cd agellic-lite-install
node install.mjs
```

### Windows (PowerShell)

```powershell
Invoke-WebRequest `
  https://github.com/Agellic-Commerce/agellic-lite/releases/latest/download/agellic-lite.zip `
  -OutFile agellic-lite.zip
Expand-Archive agellic-lite.zip agellic-lite-install
cd agellic-lite-install
node install.mjs
```

The installer:

- Prompts interactively for your **Keepa API key** and **Tokens per minute**
  (the TPM prompt accepts blank to auto-detect your rate on first use). You can
  also pass them as flags: `--keepa-key <key>`, `--tpm <integer>`.
- **Probes the server with your key** before writing any config: if the key is
  wrong, you see the error immediately instead of on the first tool call.
- Copies the server tree to the **canonical bin path** for your OS (see below)
  using a staging directory and atomic renames.
- Writes your key to the per-machine cache at
  `~/.agellic-lite/credentials-lite.json` so a subsequent Claude Desktop
  install can leave its Keepa field blank.
- Merges an `mcpServers.agellic-lite` entry into `~/.claude.json`.

Restart Claude Code (`/restart` or quit the process) to pick up the new MCP
server. The 8 Agellic Lite tools appear in the next session.

If you already configured Agellic Lite in Claude Desktop on this machine, run
`node install.mjs --non-interactive`: the installer reads your key from
`~/.agellic-lite/credentials-lite.json` and skips the prompts entirely.

### Canonical bin paths

| OS | Path |
|---|---|
| macOS | `~/Library/Application Support/Agellic-Lite/` |
| Windows | `%LOCALAPPDATA%\Agellic-Lite\` |
| Linux | `$XDG_DATA_HOME/agellic-lite/` (typically `~/.local/share/agellic-lite/`) |

The directory contains `server.js` + its runtime dependencies +
`version.json` (install metadata).

---

## Upgrade

### Claude Desktop

Drag the new `agellic-lite.mcpb` into Settings, Extensions (or **Advanced
settings**, **Install extension**, if there's no drop target). CD prompts for
your Keepa key again on the first launch after upgrade: paste it, or leave
blank to pick up from the credential cache.

### Claude Code

```bash
node install.mjs --upgrade
```

Re-reads your Keepa key from `~/.claude.json` (or the cache if the config was
wiped), so you don't need to re-pass `--keepa-key` on every upgrade. Pass it
explicitly only to rotate the key.

The upgrade uses a sibling staging directory and atomic renames: an
interrupted upgrade leaves the previous install intact, never half-installed.
It also clears cached lookups, finder and code results, and the job queue (your
key and rate-limit state are kept), so a stale result id from before the
upgrade re-runs cleanly.

---

## Uninstall

### Claude Desktop

Settings, Extensions, click the Agellic Lite extension, remove.

### Claude Code

Three levels of removal:

**Config only** (default): removes the `mcpServers.agellic-lite` entry from
`~/.claude.json`, leaves the binary and data dir in place:

```bash
node install.mjs --uninstall
```

**Config + binary**: also removes the binary at the canonical bin path. Data
dir preserved:

```bash
node install.mjs --uninstall --remove-bin
```

**Full purge**, removes everything: config entry, binary, AND
`~/.agellic-lite/` (credential cache, cached products, logs, jobs):

```bash
node install.mjs --uninstall --purge
```

The full purge **refuses by default if the Claude Desktop extension is still
installed**: the data dir is shared between hosts, and purging while CD is
active would corrupt CD's runtime state. Uninstall CD first, or pass `--force`
to override (only do this if you understand the consequences).

### Uninstall gotchas

**Lost your `install.mjs`?** The uninstaller ships only inside the release
archive; there's no separate download. If you deleted the unzipped
`agellic-lite-install/` folder, you have nothing to run `--uninstall` with.
Either re-download `agellic-lite.zip` from the
[latest release](https://github.com/Agellic-Commerce/agellic-lite/releases/latest),
unzip it, and run `node install.mjs --uninstall` (add `--remove-bin` or
`--purge`) from there, or remove Agellic Lite by hand:

1. Delete the `mcpServers.agellic-lite` entry from `~/.claude.json`.
2. Delete the canonical bin directory: macOS
   `~/Library/Application Support/Agellic-Lite/`, Windows
   `%LOCALAPPDATA%\Agellic-Lite\` (see
   [Canonical bin paths](#canonical-bin-paths)).
3. Optionally `rm -rf ~/.agellic-lite` to remove the shared data dir (see
   [Where files live](#where-files-live)).

**Quit Claude Desktop and Claude Code before `--purge`.** The purge deletes
`~/.agellic-lite/`, but a server process that's still running recreates
`~/.agellic-lite/jobs/` on its next job-queue tick, so the data dir can
reappear seconds after the purge reports success. On macOS a Claude Desktop
extension can keep running from memory even after you remove it in Settings, so
quit Claude Desktop fully (not just close the window). If you already purged
and `~/.agellic-lite/` came back, quit both hosts and `rm -rf ~/.agellic-lite`
once more.

---

## Where files live

| Location | What lives there |
|---|---|
| `~/.claude.json` | `mcpServers.agellic-lite` entry (Claude Code only): Keepa key + TPM in the env block, mode 0600 |
| `~/.agellic-lite/credentials-lite.json` | Shared per-machine credential cache (mode 0600). Both hosts read; written after every successful server boot AND after every successful installer probe |
| Canonical bin path | `server.js` + dependencies + `version.json` (Claude Code only) |
| CD extension dir | Same server tree, managed by Claude Desktop (Claude Desktop only) |
| `~/.agellic-lite/` | Shared data dir: credential cache, cached products, result sets, token bucket state, job queue, logs |

The data dir can be overridden via the `AGELLIC_LITE_DATA_DIR` environment
variable. The bin paths and CD extension paths are platform conventions and not
configurable. Agellic Lite never reads the full Agellic server's `~/.agellic-mcp`
data dir, and it refuses to run if pointed at one.

---

## Logs

Per-day rotated logs at `<dataDir>/logs/agellic-<YYYY-MM-DD>.log`. Keepa API
keys are redacted from logs.

Two env vars control log volume:

- `AGELLIC_LOG_LEVEL`: `error | warn | info | debug` (default `info`).
- `AGELLIC_LOG_DEBUG`: comma-list of module names to elevate to debug (e.g.
  `tool-call,token-bucket`).

The server prints its log path to stderr on boot. Claude Code surfaces this in
the MCP server status panel; Claude Desktop also writes its own per-server log
under `~/Library/Logs/Claude/` (macOS) or `%APPDATA%\Claude\Logs\` (Windows).
The filename starts `mcp-server-` and is derived from the Agellic Lite
extension name.

---

## Verifying an install

Ask Claude (in either host) to call **`check_token_balance`**. This exercises
the Keepa key load and the storage layer without spending any Keepa tokens.

If `check_token_balance` returns your current balance and refill rate, the
install is healthy.

If it fails, see [TROUBLESHOOTING.md](./TROUBLESHOOTING.md): the error message
names the specific failure mode.
