# Claude Code Status Line

A custom status line for [Claude Code](https://claude.ai/code) that shows model, context usage, effort level, git branch, and rate limit consumption (5h and 7d windows).

Based on [ClaudeCodeStatusLine](https://github.com/daniel3303/ClaudeCodeStatusLine) by daniel3303.

## What it shows

```
Claude Sonnet 4.6 1M | myproject@main (+3 -1) | 45k/200k (22%) | effort: med | 5h 18% @14:30 | 7d 42% @Sun Jun 8, 09:00 | v2.1.34
```

| Segment | Description |
|---|---|
| Model name | Active Claude model |
| `dir@branch` | Current folder and git branch (with diff stats if dirty) |
| `45k/200k (22%)` | Context window: tokens used / total (percentage) |
| `effort: med` | Current effort level (low/med/high/xhigh/max) |
| `5h 18% @14:30` | 5-hour rolling usage window, resets at shown time |
| `7d 42% @Sun Jun 8` | 7-day rolling usage window, resets at shown date/time |
| `extra $1.20/$5.00` | Extra usage credits (if enabled on your plan) |
| `v2.1.34` | Claude Code CLI version |

Colors go green → yellow → orange → red as usage increases (thresholds: 50%, 70%, 90%).

## How usage data is obtained

Claude Code passes a JSON payload to the script via stdin after each response. This payload includes:

- `rate_limits.five_hour.used_percentage` and `resets_at` — primary source
- `rate_limits.seven_day.used_percentage` and `resets_at` — primary source

When those values are zero or missing (which can happen due to API hiccups), the script falls back to calling `https://api.anthropic.com/api/oauth/usage` directly using your OAuth token. Results are cached in `/tmp/claude/` for 60 seconds to avoid hammering the API.

Rate limit data is only available for **Claude.ai Pro and Max** subscribers.

## Prerequisites

- `bash` (4+)
- `jq`
- `curl`
- Claude Code CLI installed and authenticated

## Installation

### 1. Copy the script

```bash
cp statusline.sh ~/.claude/statusline.sh
chmod +x ~/.claude/statusline.sh
```

### 2. Configure Claude Code

Add the following to `~/.claude/settings.json`:

```json
{
  "statusLine": {
    "type": "command",
    "command": "/home/YOUR_USER/.claude/statusline.sh"
  }
}
```

Use an absolute path — `~` is not expanded inside `settings.json`.

### 3. Restart Claude Code

The status line is loaded at startup. Close and reopen a Claude Code session to see it.

## Multiple Claude Code instances (advanced)

If you run multiple Claude Code instances with different config directories (e.g. one per client or context), you need a thin wrapper so each instance points to the correct `CLAUDE_CONFIG_DIR`.

**Wrapper script** (`~/.claude/statusline-command.sh`):
```bash
#!/usr/bin/env bash
export CLAUDE_CONFIG_DIR=/home/YOUR_USER/.claude
exec bash /home/YOUR_USER/.claude/statusline.sh
```

Then in `~/.claude/settings.json`:
```json
{
  "statusLine": {
    "type": "command",
    "command": "/home/YOUR_USER/.claude/statusline-command.sh"
  }
}
```

The script reads `CLAUDE_CONFIG_DIR` to locate the credentials file for OAuth fallback, and to read the `effortLevel` setting from `settings.json`.

## Options

| Environment variable | Default | Description |
|---|---|---|
| `CLAUDE_CONFIG_DIR` | `~/.claude` | Path to Claude config directory |
| `STATUSLINE_CHECK_UPDATES` | `true` | Set to `false` to disable GitHub update checks |
| `CLAUDE_CODE_OAUTH_TOKEN` | — | Override OAuth token explicitly |

## How the script finds your OAuth token

The script tries these sources in order:

1. `CLAUDE_CODE_OAUTH_TOKEN` environment variable
2. macOS Keychain (`security` command)
3. Linux credentials file: `$CLAUDE_CONFIG_DIR/.credentials.json`
4. GNOME Keyring (`secret-tool`)

On Linux with the default Claude Code setup, source 3 is used automatically.

## Updating

The script checks for new releases on GitHub once every 24 hours. If a newer version is available, an update notice appears below the status line.

To update manually, replace `statusline.sh` with the latest version from the [releases page](https://github.com/daniel3303/ClaudeCodeStatusLine/releases) or ask Claude: `"Find my installed status bar and update it"`.

## Troubleshooting

**Status line shows `Claude` (static text)**
The script received empty stdin. This is normal during the very first startup tick before any session data exists.

**`5h -` and `7d -` placeholders**
No usage data available. Either you are on a Free plan (rate limits not exposed) or the OAuth token could not be resolved. Check that `~/.claude/.credentials.json` exists and contains `claudeAiOauth.accessToken`.

**Script not running**
Verify the path in `settings.json` is absolute and the script is executable (`chmod +x`).

**NixOS / missing `jq`**
Add `jq` and `curl` to your system packages in `nixos/modules/dev_tools.nix` and rebuild:
```bash
sudo nixos-rebuild switch --flake /etc/nixos#
```
