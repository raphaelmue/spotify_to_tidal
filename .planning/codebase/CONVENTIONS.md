# Code Conventions

**Analysis Date:** 2026-03-30

## Style Guide

**Formatting:**
- No automated formatter (no `.prettierrc`, `pyproject.toml [tool.black]`, or `biome.json` detected)
- PEP 8 style followed loosely — 4-space indentation is standard, but `sync_playlists_wrapper` uses 2-space indentation inconsistently (`src/spotify_to_tidal/sync.py` lines 352-354)
- No linter configuration detected (no `.flake8`, `mypy.ini`, or `[tool.ruff]` section)

**Line length:**
- No enforced limit; some lines in `sync.py` are quite long (e.g., line 271)

## Naming Conventions

**Files:**
- `snake_case.py` for all modules: `sync.py`, `auth.py`, `cache.py`, `tidalapi_patch.py`
- Type definition files live under `src/spotify_to_tidal/type/`

**Functions:**
- `snake_case` throughout: `open_spotify_session`, `tidal_search`, `sync_playlist`
- Private/internal helpers prefixed with `_`: `_search_for_track_in_album`, `_run_rate_limiter`, `_get_all_chunks`, `_remove_indices_from_playlist`
- Async functions use same naming — no `async_` prefix

**Variables:**
- `snake_case`: `spotify_tracks`, `tidal_playlist`, `track_match_cache`
- Constants in `UPPER_SNAKE_CASE`: `SPOTIFY_SCOPES`
- Lambda variables use short descriptive names: `track_filter`, `sanity_filter`, `my_playlist_filter`

**Classes:**
- `PascalCase`: `MatchFailureDatabase`, `TrackMatchCache`, `SpotifyTrack`, `SpotifyAlbum`

**Parameters:**
- Type annotations used on public function signatures, not always on inner functions
- Union types use `|` syntax (Python 3.10+): `tidalapi.Track | tidalapi.Album`, `int | None`

## Code Organization

**Within files:**
- Module-level imports at the top, stdlib before third-party
- Module-level constants after imports (e.g., `SPOTIFY_SCOPES` in `auth.py`)
- Private helper functions defined as closures inside the function that uses them (e.g., `_search_for_track_in_album` inside `tidal_search`)
- Singleton instances at module bottom: `failure_cache = MatchFailureDatabase()`, `track_match_cache = TrackMatchCache()` in `cache.py`
- `__all__` used in `auth.py` to declare public API

**Module structure:**
```
src/spotify_to_tidal/
├── __main__.py        # CLI entry point and argument parsing
├── auth.py            # Session opening for Spotify and Tidal
├── cache.py           # SQLite failure cache and in-memory track cache
├── sync.py            # Core sync logic (largest file, ~419 lines)
├── tidalapi_patch.py  # Monkey-patch helpers for tidalapi pagination
└── type/
    ├── __init__.py
    ├── config.py
    └── spotify.py     # TypedDict definitions for Spotify API responses
```

**Import style:**
- Relative imports within package: `from .cache import failure_cache`
- Aliased module imports for sub-packages: `from . import sync as _sync`
- Selective imports from `typing`: `from typing import Callable, List, Sequence, Set, Mapping`

## Documentation Style

**Docstrings:**
- Used selectively on public/important functions with `""" ... """` triple-quote style
- Single-line or multi-line as appropriate:
  - `src/spotify_to_tidal/cache.py`: `MatchFailureDatabase` class and its methods each have docstrings
  - `src/spotify_to_tidal/sync.py`: key async functions documented (`populate_track_match_cache`, `get_new_spotify_tracks`, `sync_playlist`, `search_new_tracks_on_tidal`)
- Private inner functions generally not documented

**Inline comments:**
- Used liberally to explain non-obvious logic (e.g., "Leaky bucket algorithm", "double interval on each retry")
- Placed above the relevant line, not inline

**Type hints:**
- Used on public function signatures throughout
- `TypedDict` subclasses in `src/spotify_to_tidal/type/spotify.py` for Spotify API shapes
- Return types annotated: `-> str`, `-> bool`, `-> tidalapi.Track | None`
- Some older-style annotations (`List[str]`) alongside newer (`list[str]`) — mixed usage

## Error Handling

**Patterns:**
- Catch specific exception types rather than bare `except`: `except spotipy.SpotifyOauthError`, `except (tidalapi.exceptions.TooManyRequests, requests.exceptions.RequestException, ...)`
- Fatal errors use `sys.exit(message)` rather than raising: `src/spotify_to_tidal/auth.py` line 27, `src/spotify_to_tidal/__main__.py` line 22
- Retry logic encapsulated in `repeat_on_request_error` in `sync.py` — retries up to 5 times with exponential back-off via a sleep schedule dict
- `assert` used for invariants in sync logic (e.g., `sync.py` line 111) — not for input validation
- Broad `except Exception as e` used in `auth.py` for session loading failures, with `print` fallback

## Commit Conventions

**Format observed from `git log --oneline -20`:**
- Free-form imperative sentences, no enforced prefix scheme (no `feat:`, `fix:`, `chore:` Conventional Commits style)
- Examples:
  - `Fix bug in pyproject.toml: use tilde operator`
  - `Don't specify point version of spotify and tidal APIs`
  - `Bump version to 1.0.6`
  - `Add another check to the spotify track sanity filter`
  - `Filter out podcast episodes (#83)` — PR number in parentheses for merged PRs
- Version bump commits follow pattern: `Bump version to X.Y.Z`
- PR merges reference issue/PR numbers with `(#NNN)` suffix

---

*Convention analysis: 2026-03-30*
