# Technology Stack

**Analysis Date:** 2026-03-30

## Languages

**Primary:**
- Python 3.10+ - All application logic (requires >= 3.10 per `pyproject.toml`)

**Type Annotations:**
- `typing` module used throughout (`TypedDict`, `Callable`, `List`, `Sequence`, `Set`, `Mapping`, `Literal`, `Optional`)
- Typed interfaces in `src/spotify_to_tidal/type/config.py` and `src/spotify_to_tidal/type/spotify.py`

## Runtime

**Environment:**
- CPython 3.10+ (tested on 3.12 in local `.venv`)

**Package Manager:**
- `pip` with `setuptools >= 61.0` build backend
- Lockfile: Not present (no `requirements.lock` or `poetry.lock`)
- Virtual environment: `.venv/` (local, not committed)

## Frameworks & Libraries

**Core Application:**
- `spotipy ~= 2.24` - Spotify Web API client (`src/spotify_to_tidal/auth.py`, `src/spotify_to_tidal/sync.py`)
- `tidalapi ~= 0.8.10` - Tidal API client (`src/spotify_to_tidal/auth.py`, `src/spotify_to_tidal/sync.py`, `src/spotify_to_tidal/tidalapi_patch.py`)
- `pyyaml ~= 6.0` - Config file parsing and session token persistence (`src/spotify_to_tidal/__main__.py`, `src/spotify_to_tidal/auth.py`)
- `tqdm ~= 4.64` - Progress bars for sync operations (`src/spotify_to_tidal/sync.py`, `src/spotify_to_tidal/tidalapi_patch.py`)
- `sqlalchemy ~= 2.0` - SQLite ORM for persistent failure cache (`src/spotify_to_tidal/cache.py`)

**Standard Library (key usage):**
- `asyncio` - Concurrent Tidal/Spotify API requests with leaky-bucket rate limiting
- `difflib.SequenceMatcher` - Fuzzy album name matching
- `unicodedata` - Unicode normalization for cross-platform track name matching
- `argparse` - CLI argument parsing
- `webbrowser` - OAuth browser launch for Tidal login

**Testing:**
- `pytest ~= 8.0` - Test runner (config: `pytest.ini`)
- `pytest-mock ~= 3.8` - Mocking support

## Build System

**Build Backend:**
- `setuptools >= 61.0` with `wheel`
- Config: `pyproject.toml`

**Entry Point:**
- `spotify_to_tidal` CLI command → `spotify_to_tidal.__main__:main`

**Install (editable):**
```bash
pip install -e .
```

**Run Tests:**
```bash
pytest
```

## Configuration

**User Config:**
- `config.yml` - Runtime configuration (loaded at startup; see `example_config.yml` for schema)
- Key fields: `spotify.client_id`, `spotify.client_secret`, `spotify.username`, `spotify.redirect_uri`, `max_concurrency`, `rate_limit`, `sync_playlists`, `excluded_playlists`, `sync_favorites_default`

**Session Persistence:**
- `.session.yml` - Auto-generated Tidal OAuth session tokens (token_type, access_token, refresh_token, session_id); read/written by `src/spotify_to_tidal/auth.py`

**Test Config:**
- `pytest.ini` - `testpaths = tests`, `--maxfail=1 --disable-warnings`

## Infrastructure

**CI/CD:**
- CircleCI - config at `.circleci/config.yml`
- Docker image: `circleci/python:3.10`
- Workflow: `setup` → `test` (runs `pytest`)

**Local Storage:**
- `.cache.db` - SQLite database for match failure tracking (managed by `sqlalchemy` in `src/spotify_to_tidal/cache.py`)
- `songs not found.txt` - Append-only log of tracks that could not be matched on Tidal

**Deployment:**
- CLI tool — runs locally, no server/container required
- No cloud deployment target detected

---

*Stack analysis: 2026-03-30*
