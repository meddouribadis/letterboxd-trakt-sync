# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Syncs Letterboxd watched films, ratings, and watchlist to Trakt. Currently Letterboxd → Trakt only (diary entries, not full user film history).

## Common Commands

```bash
# Install dependencies (uses uv)
uv sync

# Run the script
uv run letterboxd-trakt

# Dev mode with auto-reload (uses watchexec)
mise run dev

# Linting and formatting
ruff check .
ruff format .

# Type checking
mypy .

# Pre-commit hooks
pre-commit run --all-files
```

## Architecture

Entry point is `letterboxd_trakt/main.py` (`letterboxd-trakt` CLI command defined in `pyproject.toml`). The main flow:

1. **main.py** - Loads config, iterates accounts, calls sync functions. Handles scheduling via `cronsim` when `SCHEDULED=true` (Docker default).
2. **config.py** - Pydantic models (`Config`, `Account`, `TraktOAuth`). Config stored at `config.yml` (or `/config/config.yml` in Docker). Template created automatically on first run.
3. **trakt.py** - Trakt OAuth device auth flow. Handles token refresh and retries.
4. **sync.py** - Core sync logic:
   - `sync_letterboxd_diary()` - Syncs diary entries (watched + rated). Iterates backwards to handle re-ratings correctly. Tracks `last_letterboxd_diary_entry` to avoid reprocessing.
   - `sync_letterboxd_watchlist()` - Adds Letterboxd watchlist items to Trakt.
   - Uses `DRY_RUN`, `TRAKT_RATE_LIMIT` (1.5s), and `WATCH_SEARCH_RANGE_HOURS` (48h) constants at the top of the file.

## Key Dependencies

Two dependencies are custom git forks pinned to specific commits in `uv.lock`:
- `letterboxdpy` from `https://github.com/f0e/letterboxdpy`
- `pytrakt` from `https://github.com/f0e/python-pytrakt`

These are defined in `[tool.uv.sources]` in `pyproject.toml`.

## Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `IN_DOCKER` | `false` | Set automatically in Docker. Changes config path to `/config/config.yml`. |
| `SCHEDULED` | `false` (Docker: `true`) | Run on cron schedule. |
| `RUN_ON_START` | `false` | Run immediately on container start. |
| `CRON_SCHEDULE` | `0 * * * *` | Cron expression for scheduled runs. |

## Docker

Multi-platform image (amd64/arm64) built and pushed to `ghcr.io/f0e/letterboxd-trakt-sync:latest` on pushes to `main`. Config mounted at `/config`.
