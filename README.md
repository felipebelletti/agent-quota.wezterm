# Agent Quota for WezTerm

![WezTerm plugin](https://img.shields.io/badge/WezTerm-plugin-blue)

[Listed on Awesome WezTerm](https://github.com/michaelbrusegard/awesome-wezterm#ai)

A WezTerm plugin that shows Claude and Codex quota usage directly in the status bar.

It displays live usage windows returned by each provider, reset countdowns, process-aware `not running` states, compact percentage bars, and a shared cache so multiple WezTerm windows do not all refresh the same data independently.

![Agent Quota status bar sample](assets/status-sample.svg)

> **This fork** of [M-Marbouh/agent-quota.wezterm](https://github.com/M-Marbouh/agent-quota.wezterm) adds **macOS support** (reads Claude credentials from the login Keychain; no load-time crash), per-provider configurable **icons/labels**, options to **hide the 7-day window** and **hide Codex when idle**, and a **`manual` mode** that exposes `status_string()` so you can render the quota inside your own status handler.

## Features

- Claude 5-hour and 7-day utilization
- Codex utilization windows returned by the account
- Reset countdowns for both providers
- Compact 8-cell percentage bars
- Process-aware `not running` status for Claude and Codex
- **macOS + Linux**: Claude credentials read from `~/.claude/.credentials.json` or the macOS login Keychain
- Shared per-user cache in `/tmp` across WezTerm instances
- Bundled Codex helper auto-discovery (via WezTerm's plugin registry) with no manual script-path setup
- Optional Claude usage dashboard shortcut
- **`manual` mode**: render the quota yourself via `status_string()` and merge it with other status content
- Configurable status-bar side, **per-provider icons and labels**, 7-day visibility, idle-Codex visibility, polling interval, and bar glyphs

Example output:

```text
Claude: 5h ███░░░░░ 42% (2h31m)  ▪ 7d █░░░░░░░ 18% (4d12h)  |  Codex: 5h ███████░ 88% (2h10m)  ▪ 7d ███░░░░░ 32% (1d4h)
```

## Requirements

Works on Linux, macOS, and Windows.

- [WezTerm](https://wezterm.org/)
- `python3` (or `python` on Windows)
- `curl` (bundled with Windows 10/11)
- Linux/macOS: `pgrep`, `ps`, `mkdir`, `rmdir` (plus GNU `stat` on Linux; `security` is used on macOS)
- Windows: `tasklist` (bundled) for process detection
- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) installed and authenticated for Claude usage display
- [OpenAI Codex CLI](https://github.com/openai/codex) installed and authenticated for Codex usage display

If you only use one tool, the other side simply shows `not running`.

On Debian/Ubuntu, missing system tools can usually be installed with:

```bash
sudo apt install python3 curl procps coreutils
```

Claude credentials are read automatically:

- **Linux**: `~/.claude/.credentials.json`
- **macOS**: the login Keychain item `Claude Code-credentials` (read live, so it stays valid as Claude Code rotates the token)

Codex credentials are managed by the Codex CLI itself (`~/.codex/auth.json`); this plugin talks to `codex app-server` rather than reading them directly.

## Installation

Install it with WezTerm's plugin loader:

```lua
local wezterm = require("wezterm")
local config = wezterm.config_builder()

local quota = wezterm.plugin.require("https://github.com/felipebelletti/agent-quota.wezterm")

quota.apply_to_config(config)

return config
```

Restart WezTerm fully so the plugin is cloned and loaded (a plain config reload does not reload an already-cached plugin).

No extra Python path configuration is required. The plugin locates its bundled `codex-limits.py` helper automatically via WezTerm's plugin registry (works on macOS and Linux).

## Configuration

Pass options to `apply_to_config(config, opts)`:

```lua
quota.apply_to_config(config, {
  poll_interval_secs = 120,
  position = "left",
  dashboard_key = { key = "u", mods = "CTRL|SHIFT" },

  -- Per-provider prefix. Set label = "" for an icon-only prefix, or icon = ""
  -- to drop the icon. Icons accept any string, including Nerd Font glyphs.
  claude = { icon = wezterm.nerdfonts.cod_sparkle, label = "" },
  codex  = { icon = wezterm.nerdfonts.cod_code,    label = "" },

  show_seven_day = false,        -- hide the 7-day window (show only 5h)
  hide_codex_when_idle = true,   -- hide the whole Codex segment when no Codex is running
  compact = false,               -- hide reset countdowns to save space

  icons = { week = "▪" },        -- separator glyph before the 7-day window
  bars = {
    enabled = true,
    width = 8,
    full = "█",
    empty = "░",
  },
  -- codex_script = "/absolute/path/to/codex-limits.py",
})
```

Options:

- `poll_interval_secs`: refresh interval for successful reads. Default: `60`
- `position`: `"left"` or `"right"`. Default: `"right"`
- `manual`: when `true`, the plugin does not draw its own status side; instead call
  `quota.status_string(window, pane)` from your own handler (see [Manual rendering](#manual-rendering)). Default: `false`
- `dashboard_key`: opens the Claude usage dashboard. Default: `CTRL+SHIFT+U`
- `claude.icon` / `claude.label`: prefix for the Claude segment. Either may be `""` to hide it. Defaults: `"⚡"` / `"Claude:"`
- `codex.icon` / `codex.label`: prefix for the Codex segment. Either may be `""` to hide it. Defaults: `"✦"` / `"Codex:"`
- `show_seven_day`: show the 7-day usage window. Default: `true`
- `hide_codex_when_idle`: hide the entire Codex segment (and its separator) when no Codex is running. Default: `false`
- `compact`: hide reset countdowns to shrink the status-bar footprint. Default: `false`
- `icons.week`: separator glyph before the 7-day / secondary window
- `bars.enabled`: show compact percentage bars
- `bars.width`: number of bar cells
- `bars.full` / `bars.empty`: glyphs used for the bar
- `codex_script`: absolute path to `codex-limits.py`, overriding auto-discovery. Default: `nil` (auto-detect)

### Compact mode

Set `compact = true` to drop the reset countdowns from the status bar. Usage percentages and bars are kept, only the trailing `(2h31m)` style reset timers are hidden. This is useful on narrow terminals or when the status bar shares space with other widgets.

```lua
quota.apply_to_config(config, { compact = true })
```

### Codex script path

The plugin resolves its bundled `codex-limits.py` helper automatically. If auto-discovery fails in a custom setup, point `codex_script` at the helper directly:

```lua
quota.apply_to_config(config, {
  codex_script = "/absolute/path/to/codex-limits.py",
})
```

The path is resolved lazily on first use, so an incorrect value only affects the Codex side and never blocks Claude display.

## Windows

The plugin runs on native Windows WezTerm. Platform differences are handled automatically:

- Codex process detection uses `tasklist`; Claude process detection inspects command lines so the Chrome native host can be excluded.
- The shared cache is written to the user temp directory (`%TEMP%`) instead of `/tmp`.
- Python is invoked as `python` when `python3` is not on `PATH`.

### Process detection

Claude and Codex fetching is gated on the corresponding CLI actually running, so process detection matters. Both tools install as npm shims but launch a bundled **native** binary as a child process. Claude's Chrome native host is ignored so it does not count as Claude Code:

- Claude Code runs as `claude.exe`
- Codex runs as `codex.exe` (the `node.exe` launcher spawns the native binary from its `vendor` directory)

The image name stays the same on both x64 and arm64, so `IMAGENAME eq claude.exe` / `codex.exe` matches on either architecture. You can confirm a tool is detectable while it is running with:

```powershell
tasklist /FI "IMAGENAME eq claude.exe"
tasklist /FI "IMAGENAME eq codex.exe"
```

If either command shows no process while the CLI is open, the status bar will show `not running` for that side.

### Manual rendering

By default the plugin owns a whole status side (`set_left_status` / `set_right_status`).
If you already draw your own status bar and want the quota merged into it, set
`manual = true` and render it yourself:

```lua
local quota = wezterm.plugin.require("https://github.com/felipebelletti/agent-quota.wezterm")

-- Registers the dashboard keybind but does NOT draw a status side.
quota.apply_to_config(config, { manual = true })

wezterm.on("update-status", function(window, pane)
  local ok, s = pcall(quota.status_string, window, pane)
  if ok and s and s ~= "" then
    window:set_right_status(s) -- or merge `s` with your own content
  end
end)
```

`status_string()` returns a raw-ANSI string (the same one the plugin would draw), so it
can be concatenated with other escape-coded content.

### Brand logos (optional)

This fork bundles a small font (`fonts/AgentQuotaLogos.otf`) containing the **Claude
mark** (`U+E900`) and the **Codex/OpenAI mark** (`U+E901`), so you can use the real
logos as the segment icons instead of text or Nerd Font glyphs. They are single-color
glyphs, so they take the surrounding text color.

```lua
local quota = wezterm.plugin.require("https://github.com/felipebelletti/agent-quota.wezterm")

-- 1) Load the bundled logo font; keep it last in the fallback so your primary
--    font still defines the cell metrics:
config.font_dirs = { quota.logo_font_dir() }
config.font = wezterm.font_with_fallback({ "JetBrains Mono", quota.LOGO_FONT })

-- 2) Use the logos as icons (label = "" => icon only):
quota.apply_to_config(config, {
  claude = { icon = quota.logo.claude, label = "" },
  codex  = { icon = quota.logo.codex,  label = "" },
})
```

A full WezTerm **restart** is required the first time so the new font is scanned.
Exposed helpers: `quota.LOGO_FONT` (family name), `quota.logo.claude` / `quota.logo.codex`
(the glyph strings), and `quota.logo_font_dir()` (the bundled font directory).

> The Claude and OpenAI logos are trademarks of Anthropic and OpenAI respectively, and
> are bundled solely to identify each provider in the status bar.

## How It Works

Claude:

- reads the OAuth token from `~/.claude/.credentials.json`, or the macOS login Keychain (`Claude Code-credentials`) when that file is absent
- scopes `accessToken` and `expiresAt` to the `claudeAiOauth` block, avoiding unrelated expiry fields such as `discoveryState.expiresAt`
- calls the Anthropic OAuth usage endpoint
- preserves stale data and backs off on repeated errors
- stops trusting stale data once a reported reset boundary has already passed, and briefly shows `syncing...` until fresh data arrives

Codex:

- runs the bundled `codex-limits.py`
- the helper starts `codex app-server --listen stdio://`
- reads `account/rateLimits/read`
- displays each returned usage window using its actual duration, such as `5h` or `7d`
- omits windows that are not returned and shows them again automatically if Codex reintroduces them
- displays the reset countdown from the returned `resetsAt` timestamp

Shared cache:

- Claude and Codex each write a per-user JSON cache file in `/tmp`
- a short lock directory prevents all WezTerm instances from refreshing at once
- other windows reuse the same cached result until it expires

Status display:

- shows actual usage only when the corresponding tool is running
- colors usage as green under `50%`, yellow from `50%` to `79%`, and red at `80%` and above
- renders compact 8-cell bars by default

## Compatibility

- Targets Linux, macOS, and Windows desktop sessions running WezTerm.
- Claude credentials are read from `~/.claude/.credentials.json` (Linux/Windows) or the macOS login Keychain item `Claude Code-credentials`.
- Codex usage is read through `codex app-server --listen stdio://`, so the installed Codex CLI must support app-server rate-limit reads.
- Required command-line tools on Linux/macOS are `python3`, `curl`, `pgrep`, `ps`, `mkdir`, `rmdir` (plus GNU `stat` on Linux, or `security` on macOS); on Windows they are `python`/`python3`, `curl`, and `tasklist`.

## Known Limitations

- The plugin does not refresh Claude or Codex authentication itself; it waits for the corresponding CLI to keep credentials valid.
- Codex displays `not running` unless an interactive Codex process is attached to a terminal. Quota data may still be fetchable in the background, but the visible status remains process-aware.
- Claude usage calls are intentionally cached and retried with backoff to avoid unnecessary API pressure.
- On macOS, the first time WezTerm reads the Keychain item you may get a one-time "wezterm wants to use your keychain" prompt — choose **Always Allow**.

## Troubleshooting

- Claude shows `not running`: confirm a Claude Code process is running with `pgrep -a -x claude` (Linux) or Task Manager (Windows), excluding the `--chrome-native-host` process.
- Codex shows `not running`: open Codex in a WezTerm pane and keep that pane alive; detection uses WezTerm pane process info.
- Codex helper fails in a GUI PATH environment: run `python3 codex-limits.py` directly; the helper auto-discovers common `nvm` installs.
- Codex helper path resolution fails in a custom environment: set `WEZTERM_AGENT_QUOTA_CODEX_HELPER=/absolute/path/to/codex-limits.py` before launching WezTerm.
- Cached data looks stale: inspect or remove `/tmp/wezterm-quota-limit-"$USER"-*.json` and reload WezTerm.
- Codex helper is missing: ensure the full plugin repo was installed, not just `plugin/init.lua` by itself.

## Credit

A fork of [M-Marbouh/agent-quota.wezterm](https://github.com/M-Marbouh/agent-quota.wezterm), which is itself based on [wezterm-quota-limit](https://github.com/EdenGibson/wezterm-quota-limit) by EdenGibson.

This fork adds: macOS support (Claude credentials from the login Keychain; cross-platform helper discovery via the plugin registry; no load-time crash), per-provider configurable icons/labels, `show_seven_day` and `hide_codex_when_idle` display options, and a `manual` mode that exposes `status_string()` for custom status rendering.
