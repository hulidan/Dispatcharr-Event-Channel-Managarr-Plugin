# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

A single-file Dispatcharr plugin (`Event-Channel-Managarr/plugin.py`) that manages IPTV channel visibility by analyzing channel names against configurable hide rules and EPG data. Runs inside Dispatcharr's Django/uWSGI environment — it is not a standalone Python app.

The plugin folder (`Event-Channel-Managarr/`) is zipped for distribution; `plugin.json` declares settings schema and action buttons; `plugin.py` is the entire runtime.

## Version Bumping

```bash
python3 bump_version.py              # auto: 1.26.{day-of-year}{HHMM} UTC
python3 bump_version.py 1.26.1030900 # explicit
```

This updates `version` in both `plugin.json` and `PLUGIN_VERSION` in `plugin.py`. Always bump before committing a release — the upstream marketplace CI enforces that `plugin.json` version is strictly greater than what's on `main`.

## Releasing

```bash
git tag <version> && git push origin <version>
gh release create <version> --title "v<version>" --notes "..."
gh release upload <version> Event-Channel-Managarr.zip
```

## Upstream PR to `Dispatcharr/Plugins`

PR title **must** match `[event-channel-managarr]: <description>` exactly (the validator rejects anything else). Copy `plugin.py` and `plugin.json` into `plugins/event-channel-managarr/` (lowercase-kebab, differs from this repo's capitalization).

## Architecture

Everything lives in `Event-Channel-Managarr/plugin.py`. Key classes and their roles:

| Class / area | Role |
|---|---|
| `PluginConfig` | All constants: file paths, defaults, intervals |
| `Plugin` | Main plugin class. `fields` property dynamically injects the version-check banner into the settings UI. Action handlers (`run_now`, `dry_run`, `validate_configuration`, etc.) are methods here. |
| `ProgressTracker` | Adaptive WebSocket + log progress updates during a scan |
| `SmartRateLimiter` | Optional per-channel ORM sleep between writes (`none`/`low`/`medium`/`high`) |
| Hide rule evaluation | Pure functions that test a channel name string and return `(matched: bool, reason: str)`. Registered by tag name, dispatched from the priority list in settings. |
| Duplicate resolution | Normalizes event descriptions, groups channels sharing the same event, picks one winner per the configured strategy |
| Managed Dummy EPG | Creates/reuses an `EPGSource(source_type='dummy')` row; binds/unbinds visible channels without real EPG via `EPGData` rows keyed by `channel.uuid` |
| Scheduler | Background `threading.Thread` per uWSGI worker, coordinated via `_LAST_RUN_FILE` (JSON on disk) so each scheduled time fires exactly once across all workers. `fcntl` file lock on `_SCAN_LOCK_FILE` prevents concurrent scans. |

### Django ORM imports (available at runtime inside Dispatcharr)

```python
from apps.channels.models import Channel, ChannelProfileMembership, ChannelProfile, Stream
from apps.epg.models import ProgramData
from core.utils import send_websocket_update
```

These are **not** installable via pip — they only resolve inside Dispatcharr's Django environment.

### File locations (inside the Dispatcharr container)

| File | Purpose |
|---|---|
| `/data/event_channel_managarr_settings.json` | Settings cache |
| `/data/event_channel_managarr_results.json` | Last run results |
| `/data/event_channel_managarr_last_run.json` | Cross-worker scheduled-run history |
| `/data/event_channel_managarr_scan.lock` | `fcntl` mutex |
| `/data/event_channel_managarr_undated_first_seen.json` | Per-channel first-seen dates for `[UndatedAge]` rule |
| `/data/exports/` | CSV dry-run and applied-run reports |

### Hide rule system

Rules are strings in a comma-separated priority list (e.g. `[PastDate:0],[FutureDate:2]`). The first matching rule wins; no match → channel is shown. Some rules take a parameter after the colon (`[UndatedAge:2]`, `[PastDate:0:4h]`). The `[InactiveRegex]` and `[NoEPG]` rules delegate to separate settings fields.

### Dry run vs. live run

Dry runs never write to the DB and never create/modify the managed dummy EPG source. Both produce a CSV with `#`-prefixed summary header lines followed by per-channel rows.
