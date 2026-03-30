# Architecture

**Analysis Date:** 2026-03-30

## Overview

`spotify_to_tidal` is a CLI Python tool that mirrors Spotify playlists and favorites to Tidal. It authenticates with both services, retrieves tracks from Spotify, searches for matching tracks on Tidal using a multi-strategy matching algorithm, and writes the results to Tidal playlists. A persistent SQLite cache avoids re-searching for previously failed matches across runs.

## Core Components

**Entry Point — `src/spotify_to_tidal/__main__.py`:**
- Parses CLI arguments (`--config`, `--uri`, `--sync-favorites`)
- Reads `config.yml`
- Opens sessions for both Spotify and Tidal via `auth`
- Dispatches to `sync` functions based on arguments

**Auth — `src/spotify_to_tidal/auth.py`:**
- `open_spotify_session(config)`: Creates a `spotipy.Spotify` client using OAuth2 with PKCE redirect flow
- `open_tidal_session()`: Loads a persisted OAuth token from `.session.yml` or opens a browser login flow; saves new tokens back to `.session.yml`

**Sync Engine — `src/spotify_to_tidal/sync.py`:**
- Core matching logic: `isrc_match`, `name_match`, `artist_match`, `duration_match`, `match`
- Async fetching from both APIs with concurrency control (leaky bucket rate limiter via `asyncio.Semaphore`)
- `sync_playlist`: Full playlist sync lifecycle — fetch, diff, search, write
- `sync_favorites`: Syncs Spotify saved tracks to Tidal favorites
- Helper wrappers (`sync_playlists_wrapper`, `sync_favorites_wrapper`) provide synchronous `asyncio.run()` entry points

**Tidal API Patch — `src/spotify_to_tidal/tidalapi_patch.py`:**
- Extends `tidalapi` with missing or broken functionality
- `get_all_playlists`, `get_all_playlist_tracks`, `get_all_favorites`: Async pagination helpers
- `clear_tidal_playlist`, `add_multiple_tracks_to_playlist`: Batch playlist write operations with chunking

**Cache — `src/spotify_to_tidal/cache.py`:**
- `MatchFailureDatabase`: SQLite-backed (via SQLAlchemy) persistent store of Spotify track IDs that failed to match. Uses exponential backoff retry intervals. Singleton `failure_cache`.
- `TrackMatchCache`: In-memory dict mapping Spotify track IDs to Tidal track IDs for the current run. Singleton `track_match_cache`.

**Type Definitions — `src/spotify_to_tidal/type/`:**
- `spotify.py`: `TypedDict` types for Spotify API responses (`SpotifyTrack`, `SpotifyAlbum`, `SpotifyArtist`)
- `config.py`: `TypedDict` types for config file structure (`SpotifyConfig`, `SyncConfig`, `PlaylistConfig`)

## Data Models

**SpotifyTrack** (from `src/spotify_to_tidal/type/spotify.py`):
- `id: str`, `name: str`, `duration_ms: int`, `track_number: int`
- `artists: List[SpotifyArtist]`, `album: SpotifyAlbum`
- `external_ids: Dict[str, str]` — contains `isrc` for exact matching

**Tidal Track** (`tidalapi.Track`):
- `id: int`, `name: str`, `duration: float`, `isrc: str`
- `artists: List[tidalapi.Artist]`, `version: str`

**Cache entries:**
- `failure_cache`: SQLite table `match_failures(track_id, insert_time, next_retry)`
- `track_match_cache`: In-memory `Dict[spotify_id: str, tidal_id: int]`

**Config** (`config.yml`):
- `spotify`: credentials dict
- `sync_playlists`: optional list of `{spotify_id, tidal_id}` pairs
- `excluded_playlists`: optional list of Spotify playlist URIs
- `sync_favorites_default`, `max_concurrency`, `rate_limit`

## Control Flow

**Playlist sync flow:**
1. `__main__.main()` reads config and opens sessions
2. Playlist mappings (Spotify → Tidal) determined by config mode: explicit config list, `--uri` arg, or all user playlists
3. `sync_playlist` called per playlist pair:
   a. Fetch all Spotify tracks asynchronously in paginated chunks
   b. Fetch existing Tidal playlist tracks asynchronously
   c. `populate_track_match_cache`: build in-memory match map from existing Tidal tracks
   d. `search_new_tracks_on_tidal`: async concurrent search for unmatched tracks against Tidal API (rate-limited via leaky bucket)
   e. Diff old vs. new Tidal track ID lists; append-only if possible, else clear-and-rewrite
4. Unmatched tracks logged to `songs not found.txt` and stored in `failure_cache`

**Track matching strategy (in order):**
1. ISRC exact match (when available)
2. Fuzzy match: duration within 2s + name substring match + artist intersection — with Unicode normalization fallback

## Design Patterns

- **Procedural with module-level singletons**: No classes for business logic; functions operate on passed sessions and module-level cache singletons
- **Async I/O with thread offloading**: `asyncio.to_thread` wraps blocking SDK calls; `tqdm.asyncio.gather` for parallel fetches with progress display
- **Leaky bucket rate limiting**: Custom semaphore-based rate limiter in `search_new_tracks_on_tidal`
- **Retry with exponential backoff**: `repeat_on_request_error` wraps all API calls; up to 5 retries with sleep schedule `{5:1, 4:10, 3:60, 2:300, 1:600}` seconds
- **Monkey-patching external library**: `tidalapi_patch.py` adds async pagination and missing bulk-write methods to `tidalapi` objects
- **TypedDict for external API shapes**: `type/` module defines typed wrappers for Spotify JSON without runtime overhead

## Entry Points

**CLI command:**
```bash
spotify_to_tidal --config config.yml
spotify_to_tidal --uri spotify:playlist:<id>
spotify_to_tidal --sync-favorites
spotify_to_tidal --no-sync-favorites
```

**Python module:**
```bash
python -m spotify_to_tidal --config config.yml
```

**Registered script** (`pyproject.toml`):
- `spotify_to_tidal = "spotify_to_tidal.__main__:main"`

---

*Architecture analysis: 2026-03-30*
