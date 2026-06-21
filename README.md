# Agent Quota for WezTerm

![WezTerm plugin](https://img.shields.io/badge/WezTerm-plugin-blue)

[Listed on Awesome WezTerm](https://github.com/michaelbrusegard/awesome-wezterm#ai)

A WezTerm plugin that shows Claude and Codex quota usage directly in the status bar.

It displays live 5-hour and 7-day usage windows, reset countdowns, process-aware `not running` states, compact percentage bars, and a shared cache so multiple WezTerm windows do not all refresh the same data independently.

![Agent Quota status bar sample](assets/status-sample.svg)

> **This fork** of [M-Marbouh/agent-quota.wezterm](https://github.com/M-Marbouh/agent-quota.wezterm) adds **macOS support** (reads Claude credentials from the login Keychain; no load-time crash), per-provider configurable **icons/labels**, options to **hide the 7-day window** and **hide Codex when idle**, and a **`manual` mode** that exposes `status_string()` so you can render the quota inside your own status handler.

## Features

- Claude 5-hour and 7-day utilization
- Codex 5-hour and 7-day utilization
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

Works on Linux and macOS.

- [WezTerm](https://wezterm.org/)
- `python3`
- `curl`
- `pgrep`, `ps`, `mkdir`, `rmdir` (and GNU `stat` on Linux; `security` is used on macOS)
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

  icons = { week = "▪" },        -- separator glyph before the 7-day window
  bars = {
    enabled = true,
    width = 8,
    full = "█",
    empty = "░",
  },
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
- `icons.week`: separator glyph before the 7-day window
- `bars.enabled`: show compact percentage bars
- `bars.width`: number of bar cells
- `bars.full` / `bars.empty`: glyphs used for the bar

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

## How It Works

Claude:

- reads the OAuth token from `~/.claude/.credentials.json`, or the macOS login Keychain (`Claude Code-credentials`) when that file is absent
- calls the Anthropic OAuth usage endpoint
- preserves stale data and backs off on repeated errors
- stops trusting stale data once a reported reset boundary has already passed, and briefly shows `syncing...` until fresh data arrives

Codex:

- runs the bundled `codex-limits.py`
- the helper starts `codex app-server --listen stdio://`
- reads `account/rateLimits/read`

Shared cache:

- Claude and Codex each write a per-user JSON cache file in `/tmp`
- a short lock directory prevents all WezTerm instances from refreshing at once
- other windows reuse the same cached result until it expires

Status display:

- shows actual usage only when the corresponding tool is running
- colors usage as green under `50%`, yellow from `50%` to `79%`, and red at `80%` and above
- renders compact 8-cell bars by default

## Compatibility

- Targets Linux and macOS desktop sessions running WezTerm.
- Claude credentials are read from `~/.claude/.credentials.json` (Linux) or the macOS login Keychain item `Claude Code-credentials`.
- Codex usage is read through `codex app-server --listen stdio://`, so the installed Codex CLI must support app-server rate-limit reads.
- Required command-line tools: `python3`, `curl`, `pgrep`, `ps`, `mkdir`, `rmdir`; plus GNU `stat` on Linux, or `security` on macOS.

## Known Limitations

- The plugin does not refresh Claude or Codex authentication itself; it waits for the corresponding CLI to keep credentials valid.
- Codex displays `not running` unless an interactive Codex process is attached to a terminal. Quota data may still be fetchable in the background, but the visible status remains process-aware.
- Claude usage calls are intentionally cached and retried with backoff to avoid unnecessary API pressure.
- On macOS, the first time WezTerm reads the Keychain item you may get a one-time "wezterm wants to use your keychain" prompt — choose **Always Allow**.
- Windows is not a tested target.

## Troubleshooting

- Claude shows `not running`: confirm `pgrep -x claude` returns a process.
- Codex shows `not running`: open Codex in a WezTerm pane and keep that pane alive; detection uses WezTerm pane process info.
- Codex helper fails in a GUI PATH environment: run `python3 codex-limits.py` directly; the helper auto-discovers common `nvm` installs.
- Codex helper path resolution fails in a custom environment: set `WEZTERM_AGENT_QUOTA_CODEX_HELPER=/absolute/path/to/codex-limits.py` before launching WezTerm.
- Cached data looks stale: inspect or remove `/tmp/wezterm-quota-limit-"$USER"-*.json` and reload WezTerm.
- Codex helper is missing: ensure the full plugin repo was installed, not just `plugin/init.lua` by itself.

## Credit

A fork of [M-Marbouh/agent-quota.wezterm](https://github.com/M-Marbouh/agent-quota.wezterm), which is itself based on [wezterm-quota-limit](https://github.com/EdenGibson/wezterm-quota-limit) by EdenGibson.

This fork adds: macOS support (Claude credentials from the login Keychain; cross-platform helper discovery via the plugin registry; no load-time crash), per-provider configurable icons/labels, `show_seven_day` and `hide_codex_when_idle` display options, and a `manual` mode that exposes `status_string()` for custom status rendering.
