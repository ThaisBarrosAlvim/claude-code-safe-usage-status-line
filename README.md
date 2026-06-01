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

## Security improvements over the original

This script is a hardened fork of [daniel3303/ClaudeCodeStatusLine](https://github.com/daniel3303/ClaudeCodeStatusLine). The following issues were fixed:

**OAuth token exposed in process list**
The original passes the token as a `-H "Authorization: Bearer $token"` argument to `curl`, making it visible to any process that reads `/proc/<pid>/cmdline` (e.g. `ps aux`) during the fraction of a second curl runs. This fork passes all headers via `curl --config -` (stdin heredoc), so the token never appears as a command-line argument.

**Cache directory world-readable**
The original creates `/tmp/claude/` with default permissions (`755`), allowing any local user or daemon to read the cached usage data. This fork creates it with `700` (owner-only), and also fixes an existing `/tmp/claude` directory if it was already created with the wrong permissions.

**Update checker removed**
The original polls `https://api.github.com/repos/daniel3303/ClaudeCodeStatusLine/releases/latest` every 24 hours to check for new versions. Beyond the unnecessary outbound call to a third-party repo, this also leaks metadata about your Claude Code version and usage patterns. This fork removes the feature entirely — updates are managed through git.

## Options

| Environment variable | Default | Description |
|---|---|---|
| `CLAUDE_CONFIG_DIR` | `~/.claude` | Path to Claude config directory |
| `CLAUDE_CODE_OAUTH_TOKEN` | — | Override OAuth token explicitly |

## How the script finds your OAuth token

The script tries these sources in order:

1. `CLAUDE_CODE_OAUTH_TOKEN` environment variable
2. macOS Keychain (`security` command)
3. Linux credentials file: `$CLAUDE_CONFIG_DIR/.credentials.json`
4. GNOME Keyring (`secret-tool`)

## Troubleshooting

**Status line shows `Claude` (static text)**
The script received empty stdin. This is normal during the very first startup tick before any session data exists.

**`5h -` and `7d -` placeholders**
No usage data available. Either you are on a Free plan (rate limits not exposed) or the OAuth token could not be resolved. Check that `~/.claude/.credentials.json` exists and contains `claudeAiOauth.accessToken`.

**Script not running**
Verify the path in `settings.json` is absolute and the script is executable (`chmod +x`).
