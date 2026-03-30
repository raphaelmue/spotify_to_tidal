# Project Structure

**Analysis Date:** 2026-03-30

## Directory Layout

```
spotify_to_tidal/
├── src/
│   └── spotify_to_tidal/         # Main package
│       ├── __main__.py            # CLI entry point and argument parsing
│       ├── auth.py                # Spotify and Tidal OAuth session management
│       ├── cache.py               # Match failure DB (SQLite) and in-memory track cache
│       ├── sync.py                # Core sync logic, track matching, async search
│       ├── tidalapi_patch.py      # Async pagination and bulk write extensions for tidalapi
│       └── type/                  # TypedDict definitions for API shapes
│           ├── __init__.py
│           ├── config.py          # Config file types (SpotifyConfig, SyncConfig)
│           └── spotify.py         # Spotify API response types (SpotifyTrack, SpotifyAlbum)
├── tests/
│   ├── conftest.py                # Adds src/ to sys.path for test imports
│   └── unit/
│       ├── __init__.py
│       ├── test_auth.py           # Unit tests for auth module
│       └── test_cache.py          # Unit tests for cache module
├── .circleci/
│   └── config.yml                 # CI pipeline configuration
├── src/spotify_to_tidal.egg-info/ # Build metadata (generated, not edited)
├── .venv/                         # Virtual environment (not committed)
├── .cache.db                      # SQLite failure cache (runtime artifact)
├── .session.yml                   # Persisted Tidal OAuth tokens (runtime artifact)
├── config.yml                     # User's local config (not committed)
├── example_config.yml             # Template config for new users
├── songs not found.txt            # Output log of unmatched tracks (runtime artifact)
├── pyproject.toml                 # Package metadata, dependencies, entry points
├── pytest.ini                     # Pytest configuration
├── readme.md                      # Project documentation
└── LICENSE
```

## Key Files

| File | Role |
|---|---|
| `src/spotify_to_tidal/__main__.py` | CLI argument parsing, session init, dispatch to sync |
| `src/spotify_to_tidal/sync.py` | All sync logic: matching, async search, playlist diffing, rate limiting |
| `src/spotify_to_tidal/auth.py` | OAuth flows for both Spotify (spotipy) and Tidal (tidalapi) |
| `src/spotify_to_tidal/cache.py` | `MatchFailureDatabase` (SQLite) and `TrackMatchCache` (in-memory) singletons |
| `src/spotify_to_tidal/tidalapi_patch.py` | Async wrappers and bulk-write helpers missing from tidalapi |
| `src/spotify_to_tidal/type/spotify.py` | TypedDict types for Spotify track/album/artist JSON |
| `src/spotify_to_tidal/type/config.py` | TypedDict types for `config.yml` structure |
| `pyproject.toml` | Declares package, Python >=3.10 requirement, all dependencies, CLI entry point |
| `example_config.yml` | Reference config showing all available options with comments |
| `pytest.ini` | Test runner config: `testpaths = tests`, `--maxfail=1` |

## Module Organization

The package is a flat collection of focused modules — no sub-packages except `type/`:

- **`auth`**: Isolated OAuth concerns. No business logic. Reads `config` dict, writes `.session.yml`.
- **`cache`**: Two persistence strategies in one module. `MatchFailureDatabase` persists across processes; `TrackMatchCache` is per-run only. Both exposed as module-level singletons imported by `sync`.
- **`sync`**: Largest module (~420 lines). Contains all matching algorithms, async fetch logic, rate limiter, and playlist diffing. Imports cache singletons directly.
- **`tidalapi_patch`**: Pure extension of `tidalapi` objects. No imports from other project modules.
- **`type/`**: Purely declarative TypedDict module. No logic. Imported by `sync` as `t_spotify`.
- **`__main__`**: Thin orchestration layer only. Reads config, calls `auth`, dispatches to `sync`.

## Naming Conventions

**Files:**
- `snake_case.py` for all modules
- `test_<module>.py` for test files mirroring module names

**Functions:**
- `snake_case` for all functions
- `_leading_underscore` for private/internal helpers (e.g., `_search_for_track_in_album`, `_run_rate_limiter`, `_get_all_chunks`)
- `get_`, `sync_`, `open_`, `pick_`, `populate_` prefixes for clarity

**Classes:**
- `PascalCase`: `MatchFailureDatabase`, `TrackMatchCache`, `SpotifyTrack`, `SpotifyConfig`

**Config keys:**
- `snake_case` in YAML (e.g., `sync_favorites_default`, `max_concurrency`)

## Where to Add New Code

**New sync operation (e.g., sync albums):**
- Implementation: `src/spotify_to_tidal/sync.py`
- New CLI flag: `src/spotify_to_tidal/__main__.py`

**New Tidal API wrapper (pagination/bulk write):**
- Implementation: `src/spotify_to_tidal/tidalapi_patch.py`

**New Spotify API type:**
- Definition: `src/spotify_to_tidal/type/spotify.py`

**New config option:**
- Type definition: `src/spotify_to_tidal/type/config.py`
- Documentation: `example_config.yml`

**New tests:**
- Unit tests: `tests/unit/test_<module>.py`
- No integration test directory exists yet

---

*Structure analysis: 2026-03-30*
