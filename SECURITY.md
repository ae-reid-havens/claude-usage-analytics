# Security Policy

## Supported Versions

Only the latest release published to the VS Code Marketplace is supported with security updates.

## What this extension touches

Claude Usage Analytics runs fully offline and makes no network calls, but it does read
local files that may be sensitive:

- `~/.claude/.credentials.json` (or the macOS Keychain entry `Claude Code-credentials`) — read-only, used to display your subscription tier
- `~/.claude/projects/*/` — conversation history, analyzed locally
- `~/.claude/analytics.db` — usage statistics written by this extension (no conversation content)

If GitHub Gist sync is enabled, only `analytics.db` (usage statistics) is uploaded, to a
private Gist using a token you provide.

## Reporting a vulnerability

Please report vulnerabilities privately via
[GitHub Security Advisories](https://github.com/AnalyticEndeavorsUser/claude-usage-analytics/security/advisories/new)
rather than opening a public issue. You should receive a response within a few business days.
