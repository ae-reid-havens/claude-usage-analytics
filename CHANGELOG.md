# Changelog

All notable changes to the Claude Usage Analytics extension will be documented in this file.

## [1.1.15] - 2026-08-14

### Added
- **"Claude Analytics: Scan History (Keep Existing)" command** - Runs the JSONL backfill and merges the results, never truncating. This is the same operation the first-run "Scan History" prompt offers, but that prompt fires once per install and can never be shown again, so the only reachable backfill afterwards was "Recalculate Historical Costs", which calls `truncateAllData()` first. Claude Code prunes old session files, so on any install older than that retention window a truncate-and-rebuild silently destroys history that survives nowhere but the database. This command is what fills Request Types and Longest Session, whose tables (`work_classification_daily`, `sessions`) are written only by the backfill.
- **Claude Opus 5 pricing** - Added `claude-opus-5` to the model pricing table at $5/$25 per MTok (verified against Anthropic pricing docs). Cache rates for the shared `opus_new` tier are unchanged and still correct: $0.50/MTok cache read, $6.25/MTok 5-minute cache write.

### Fixed
- **Dashboard showed zero for everything** - `getUsageData()` returned an empty result set as soon as `~/.claude/stats-cache.json` was missing, and current Claude Code versions no longer write that file. The early return fired before the SQLite read and before the live-scan merge, so a fully populated `analytics.db` was never consulted. The stats-cache is now an optional enrichment source: when it is absent, daily history, model breakdown, date range, and today's figures are built from `analytics.db` and today's JSONL scan instead. The stats-cache path is unchanged for anyone still on a Claude Code version that writes it.
- **Account total, token breakdown, and activity patterns still read zero after the first fix** - The account total card and its token breakdown both default to the API view, which is sourced purely from `stats-cache.json`, so the whole panel showed 0 while `accountTotalCalculated` held the real SQLite figures beside it. The calculated totals are now mirrored into the API and default views when there is no API data to show; the calculated/API toggle is unchanged wherever a stats-cache exists. Peak Hour, Night Owl, and Early Bird read `statsCache.hourCounts` and `conversation-stats-cache.json`, neither of which Claude Code still writes, and now fall back to the `hourly_distribution` table.
- **Model breakdown from SQLite is more accurate than the cache estimate** - The SQLite fallback sums real per-type token counts from `model_usage` rather than applying the 30/10/50/10 input/output/cache-read/cache-write split the stats-cache path has to assume.
- **Historical day costs were shown at the rates in effect when they were recorded** - Days merged in from SQLite now prefer a cost recomputed from stored token counts at current prices, matching how cache-sourced days were already handled. Without this, a pricing correction never reached days already written to the database.
- **Opus 5 was labelled "Opus" in the model breakdown** - `formatModelName()` extracted a MAJOR-MINOR pair, which major-only ids such as `claude-opus-5`, `claude-sonnet-5`, and `claude-fable-5` do not have, so they collapsed to a bare family name. It now falls back to a 1-2 digit version immediately after the family name; the length bound keeps a trailing date stamp (`claude-3-opus-20240229`) from being read as a version. Fable and Mythos are also recognised as families now, with their own chart colour, instead of falling through to the raw model id.
- **Opus 5 usage was costed at $15/$75 instead of $5/$25** - `getPricingForModel()` (in `dataProvider.ts` and `database.ts`) matched the `opus_new` tier on the substrings `4-5`/`4-6`/`4-7`/`4-8`. None of those appear in `claude-opus-5`, so it fell through to `opus_legacy` and every Opus 5 token was billed at the Claude 3 Opus rate - a 3x overstatement of cost. The tier now also matches `opus-5` explicitly; a bare `5` is not used because it would collide with `4-5`.
- **Unrecognised Opus model IDs defaulted to Claude 3 Opus rates** - The JSON-keyed lookup shared by `scan-today.js`, `backfill-jsonl.js`, and `backfillManager.ts` fell back to `claude-3-opus-20240229` ($15/$75) for any `opus` ID without an exact table entry, which is what carried the Opus 5 error into today's live stats. Every legacy Opus already has an explicit entry, so an unmatched ID is far more likely to be a newly released model; the fallback is now the current Opus tier ($5/$25).

### Changed
- **"Recalculate Historical Costs" warning now states what is actually at risk** - The old confirmation said it would clear data and rescan, which reads like a rebuild. It does not mention that Claude Code prunes old JSONL, so any day older than what remains on disk is lost rather than recalculated. The warning now says so and points at the non-destructive command instead.
- **Marketplace virus check rejected the 1.1.15 upload; package trimmed to what the extension actually loads** - The verification log gave no detail beyond "Extension failed Virus check", and the file list was byte-for-byte identical to the 1.1.14 package that passed in July, so the trigger was an existing artifact rather than a new one. Removed everything with no runtime purpose: the unused `keytar` dependency (a prebuilt 707 KB native binary from 2022, imported nowhere in the source), `__pycache__` bytecode, a stray Playwright capture, CI config, `build.bat`, and sql.js's debug builds, web-worker variants, and three nested release zips. An archive inside the VSIX is the specific thing a scanner cannot see into. The package went from 77 files and 8.65 MB to 32 files and 795 KB, and now contains no binary of any kind.
- **`.vscodeignore` node_modules rule inverted to an allowlist** - vsce will not re-exclude a path once a parent glob has been negated, so `!node_modules/sql.js/**` followed by exclusions silently shipped the entire package. Only the files sql.js needs are allowed back in. Note that `database.ts` requires `sql.js/dist/sql-asm.js`, the pure-JS asm.js build, not the wasm one; an allowlist naming the wasm files instead builds and passes tests against the source tree, then fails for every user. Verify changes to that list by extracting the built VSIX and loading `out/dataProvider.js` from the extracted copy.
- **Sonnet 5 $2/$10 is now the standard price** - Anthropic has confirmed that the rate announced as introductory pricing through 2026-08-31 is now standard, and the increase to $3/$15 scheduled for 2026-09-01 will not happen. This supersedes the note in 1.1.14: the `sonnet_5` tier and its `sonnet-5` branch are permanent and should not be removed. Dated comments in `dataProvider.ts` and `database.ts` updated accordingly.

## [1.1.14] - 2026-07-08

### Added
- **Claude Sonnet 5 pricing** - Added `claude-sonnet-5` to the model pricing table at $2/$10 per MTok (introductory pricing verified against Anthropic pricing docs). `getPricingForModel()` (in `dataProvider.ts` and `database.ts`) now resolves `sonnet-5` via a dedicated `sonnet_5` tier that is checked **before** the generic `sonnet` branch — without it, `claude-sonnet-5` would match `sonnet` and be overcosted at $3/$15.

### Note
- **Sonnet 5 introductory pricing ends 2026-08-31.** On 2026-09-01 Sonnet 5 reverts to the standard $3/$15 (identical to the generic `sonnet` tier). At that point the temporary `sonnet_5` tier and its `sonnet-5` branch should be removed so it falls through to `sonnet`, and `modelPricing.json` updated to $3/$15. Resolves tools-watchlist issue #629.

## [1.1.13] - 2026-06-12

### Added
- **Claude Fable 5 / Mythos 5 pricing** - Added `claude-fable-5` and `claude-mythos-5` to the model pricing table at $10/$50 per MTok (verified against Anthropic pricing docs). Previously Fable 5 usage fell through to the `default` Sonnet rate ($3/$15), undercosting by more than 3x. `getPricingForModel()` (in `dataProvider.ts` and `database.ts`) now resolves `fable-5` / `mythos-5` to the new tier.
- **`claude-haiku-4-5` alias** - Added the non-dated Haiku 4.5 model ID alongside the dated `claude-haiku-4-5-20251001` entry so usage logged under the alias resolves to Haiku pricing instead of the Sonnet default.

### Fixed
- **CHANGELOG dates** - Corrected the 1.1.6 and 1.1.7 release dates from 2025 to 2026 (they shipped between 1.1.5 on 2025-12-30 and 1.1.8 on 2026-01-10).
- **Repository links** - README Issues/Releases links now point at the canonical `AnalyticEndeavorsUser/claude-usage-analytics` instead of relying on GitHub's rename redirect from `analyticendeavors/`.
- **Claude Code docs link** - Updated the installation guide link to the current docs home (`code.claude.com/docs`).
- **Copyright years** - README and LICENSE updated to 2024-2026.

## [1.1.12] - 2026-06-01

### Added
- **Opus 4.8 pricing** - Added `claude-opus-4-8` to model pricing table at $5/$25 per MTok (verified against Anthropic pricing docs). `getPricingForModel()` (in `dataProvider.ts` and `database.ts`) now matches `4-8` alongside `4-5`/`4-6`/`4-7` for the new Opus pricing tier.

## [1.1.11] - 2026-05-06

### Added
- **Opus 4.7 pricing** - Added `claude-opus-4-7` to model pricing table at $5/$25 per MTok (verified against Anthropic pricing docs).
- **Accessibility (WCAG 2.1 AA) improvements** - ARIA tab pattern on dashboard tabs (`role="tablist"`, `role="tab"`, `role="tabpanel"`, `aria-selected`); SVG charts (bar, heatmap, pie) now have `role="img"` with descriptive `aria-label`; chart and data-source toggle buttons expose `aria-label` and `aria-pressed`; personality bars expose `role="progressbar"` with `aria-valuenow/min/max`; achievement badges and footer links have screen-reader-friendly `aria-label`s; section titles expose `role="heading" aria-level="2"`; visible `:focus-visible` outlines on tabs, toggles, and buttons; `prefers-reduced-motion` honoured.

### Fixed
- **Newly-released models invisible in pie chart** - The models breakdown was built only from Claude Code's `stats-cache.json`, which lags new model IDs. The pie chart now supplements with models from today's live JSONL scan, so models like Opus 4.7 appear immediately.
- **Model name version parsing** - `formatModelName()` no longer hardcodes "Opus 4.5" / "Sonnet 4.5"; it now extracts MAJOR.MINOR from the model id (e.g. Opus 4.7, Sonnet 4.6, Sonnet 3.5).
- **Cache savings used Sonnet rate for all models** - The Cache Savings stat now accumulates per-model savings (`input - cacheRead`) instead of a flat Sonnet estimate, so Opus-heavy users see accurate savings.
- **Today's cost showed $0.00 on extension startup** - Dashboard now hydrates `~/.claude/live-today-stats.json` synchronously on activation (when its date matches today) before the 3-second delayed scan fires.
- **Streak broke when stats-cache.json was stale** - `daysWithActivity` is now supplemented from SQLite `model_usage` records and today's live scan, so the streak no longer drops to 0 when Claude Code hasn't refreshed its cache.
- **"Scan failed" on large histories** - `tools/scan-today.js` now stats each JSONL file and skips ones whose mtime isn't today, dropping scan time from ~28s to ~1s on 4,400+ file histories. Scan execFile timeout raised from 30s to 60s as a safety margin.
- **Opus 4.7 fell through to legacy pricing** - `getPricingForModel()` (in `dataProvider.ts` and `database.ts`) now matches `4-7` alongside `4-5`/`4-6` for the new Opus pricing tier.
- **"1 msgs" pluralisation** - Longest Session stat now says "1 msg" / "N msgs".

## [1.1.10] - 2026-05-06

### Fixed
- **Subscription tier shows "N/A" on macOS** ([#7](https://github.com/AnalyticEndeavorsUser/claude-usage-analytics/issues/7)) - Claude Code on macOS stores credentials in the system Keychain (service `Claude Code-credentials`) rather than `~/.claude/.credentials.json`. The extension now reads from the Keychain via the `security` CLI on macOS, falling back to the on-disk file if no Keychain entry exists. The first read may trigger a one-time Keychain access prompt.

## [1.1.9] - 2026-04-09

### Added
- **External additions sidecar framework** - Import usage data from any external AI tool (Copilot CLI, Forge CLI, etc.) via a JSON file at `~/.claude/external-additions.json`. External data merges seamlessly into all dashboard views, streaks, and totals.

### Fixed
- **Incorrect model pricing** - Reverted Haiku 3.5 to correct $0.80/$4.00 (was wrongly changed to $1/$5). Fixed Opus 4.5 from $15/$75 to correct $5/$25. Added missing Opus 4.1 and Sonnet 4.5 model entries.
- **Haiku pricing missing from cost calculations** - Added Haiku tier pricing (4.5, 3.5, 3) to both dataProvider.ts and database.ts. Previously Haiku usage fell through to Sonnet pricing ($3/$15), overcharging by 3x or more.
- **Opus 4.5/4.6 overcharged in historical costs** - Split Opus pricing into new ($5/$25 for 4.5/4.6) vs legacy ($15/$75 for 4.1/4/3). Previously all Opus models were charged at the legacy rate.
- **README claimed open source** - Removed source code references, build-from-source instructions, and Development section. Extension is free forever but source is not distributed.
- **CHANGELOG claimed non-existent weeklyBudget feature** - Corrected to only reference `dailyBudget`
- **Broken BACKFILL_GUIDE.md link** - Replaced with reference to dashboard instructions
- **README version badge** - Updated from 1.1.5 to 1.1.9

## [1.1.8] - 2026-01-10

### Fixed
- **"Scan failed: Command failed" error on Windows** - Fixed critical bug where `process.execPath` returns VS Code's `Code.exe` instead of Node.js in the extension host environment. The scan command now correctly uses `node` from the system PATH. This reverts the problematic v1.1.6 fix that caused the regression.

## [1.1.7] - 2026-01-09

### Changed
- **Footer link updated** - Replaced YouTube link with Donate link (Buy Me a Coffee) in dashboard footer

## [1.1.6] - 2026-01-05

### Fixed
- **"Spawn node ENOENT" error on install** - Fixed error that occurred when users didn't have Node.js in their system PATH. Extension now uses VSCode's bundled Node.js runtime (`process.execPath`) instead of relying on a global `node` command.

## [1.1.5] - 2025-12-30

### Added
- **Context Overhead section** - Tokens tooltip now shows MCP servers, skills count, and tool calls under "Context Overhead" (below Cache Efficiency)
- **3 new visibility settings** - Toggle MCP status, tool calls, and skills count display independently

### Improved
- **Instant settings update** - Toggling any visibility setting now immediately refreshes the status bar (no reload required)
- **Top Language display** - Now shows comma-formatted count with label (e.g., "python - 42,600 blocks")

### Fixed
- **Realistic politeness thresholds** - Adjusted scoring for coding context (5%+ = Polite, 2%+ = Friendly, 1%+ = Neutral, <1% = All Business)
- **Clearer politeness display** - Tooltip now shows descriptive label with percentage (e.g., "Friendly (1.7%)")
- **Backfill script link** - Updated to new GitHub repository URL (analyticendeavors/claude-usage-analytics)

## [1.1.4] - 2025-12-29

### Improved
- **Enhanced Account Total tooltip** - Status bar tooltip now shows both API Total (from stats-cache.json) and Calculated Total (from SQLite + JSONL history), giving visibility into both data sources similar to the main dashboard view

## [1.1.3] - 2025-12-29

### Fixed
- **Yesterday cost showing N/A** - Fixed issue where "Yesterday" cost would incorrectly show N/A even when data existed. Added fallback logic to ensure yesterday's cost is calculated from SQLite or estimated from token data when the primary calculation returns zero.

## [1.1.2] - 2025-12-28

### Added
- **Open Settings command** - New "Claude Analytics: Open Settings" command for quick access to extension settings
- **Live config updates** - Changing refresh interval in settings now takes effect immediately without requiring a reload
- **Disable auto-refresh option** - Set refresh interval to 0 to disable auto-refresh entirely

### Fixed
- **Auto-refresh now scans live stats** - The auto-refresh interval now properly re-scans JSONL files for today's usage, fixing an issue where today's numbers stayed stale until manual refresh

### Changed
- **Refresh interval now in seconds** - Replaced `refreshIntervalMinutes` setting with `refreshIntervalSeconds` for finer control (0=disabled, 10-3600 seconds, default 900 = 15 minutes)

## [1.1.0] - 2025-12-27

### Added
- **SQLite persistence** - Usage history now preserved forever in a local SQLite database, surviving Claude Code's 30-day rolling window
- **Configurable refresh interval** - New `refreshIntervalMinutes` setting (1-60 min, default 15) to control auto-refresh frequency
- **Historical data import** - On first install, automatically imports existing data from stats-cache.json
- **Local history stats** - "Local History" totals now include full data from your local SQLite database, not just the last 30 days
- **7 new achievements** - Token Titan (1M+ tokens), $100 Club, $500 Spender, $1K Whale, Refactor Pro, Refactor King, Weekend Warrior
- **Export to CSV/JSON** - Export your usage data via dashboard button or view title menu
- **Budget tracking** - New `dailyBudget` setting with status bar color coding (green/yellow/red)
- **Cost alerts** - New `costAlertThreshold` setting triggers VS Code notifications when daily cost exceeds threshold
- **Date range filter** - Filter dashboard stats by Last 7 days, Last 30 days, This Month, or All Time
- **Session breakdown** - New section in Messages tab showing recent sessions with project, messages, tokens, and cost
- **Activity heatmap** - GitHub-style contribution calendar on Personality tab showing last 90 days of activity
- **Theme-aware colors** - All UI elements now adapt to light and dark VS Code themes
- **Backfill script** - Python script to import full Claude.ai conversation history from data export
- **Personality analytics** - Request types, sentiment tracking, and celebration moments
- **GitHub Gist sync** - Backup your analytics database to a private Gist for multi-machine sync
- **Status bar visibility settings** - 7 new settings to show/hide individual status bar items (lifetime cost, today cost, messages, tokens, personality, activity, rate limits)

### Changed
- Chart toggle buttons now use emojis (messages, cost, tokens)
- Footer includes Analytic Endeavors branding with logo and links

## [1.0.3] - 2025-12-21

### Added
- **Real-time today's cost** - Now reads directly from conversation JSONL files for accurate current-day statistics
- **Subscription tier display** - Shows Max 20x, Max, Pro, or Free tier instead of rate limit percentages
- **Improved tooltips** - All status bar widgets now show "Click to open [Tab Name]" for clarity

### Changed
- **Fully offline** - Removed all network API calls; extension operates completely locally
- **Fixed credentials reading** - Now correctly reads from `~/.claude/.credentials.json`

### Removed
- **Rate limit monitoring** - Removed Limits section and rate limit progress bars (obsolete after API changes)

## [1.0.0] - 2025-12-20

### Initial Release

First public release as a standalone VS Code extension.

### Features
- **4-Tab Dashboard** - Interactive sidebar with Overview, Cost, Messages, and Personality tabs
- **7 Status Bar Widgets** - Live statistics always visible:
  - Lifetime Cost with trend analysis
  - Today's Cost with comparisons
  - Total Messages with activity patterns
  - Token Count with cache efficiency
  - Personality Score with trait breakdown
  - Activity Stats with coding metrics
  - Subscription Tier display

- **Cost Analytics**
  - Accurate pricing using model-specific rates (Opus vs Sonnet)
  - Daily, weekly, and monthly trends
  - Cost projections and comparisons
  - Cache savings tracking

- **Personality Insights**
  - Politeness, frustration, and curiosity scores
  - Achievement badges for milestones
  - Expression style analysis
  - Mood and sentiment tracking

- **Activity Tracking**
  - Code block and line counts
  - Top languages used
  - Request type distribution (debug, feature, explain, etc.)
  - Peak hours and activity patterns

### Keyboard Shortcuts
- `Ctrl+Alt+C` (`Cmd+Alt+C` on Mac) - Show Analytics Panel
- `Ctrl+Alt+R` (`Cmd+Alt+R` on Mac) - Refresh Data

---

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).
