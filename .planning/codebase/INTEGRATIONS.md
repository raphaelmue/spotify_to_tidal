# External Integrations

**Analysis Date:** 2026-03-30

## APIs & External Services

**Spotify Web API:**
- Purpose: Source of truth — reads user playlists, playlist tracks, and saved (favorite) tracks
- SDK: `spotipy ~= 2.24`
- Key calls (in `src/spotify_to_tidal/sync.py`):
  - `spotify_session.current_user_playlists()` — list user playlists
  - `spotify_session.playlist_tracks()` — get tracks in a playlist
  - `spotify_session.current_user_saved_tracks()` — get user favorites
  - `spotify_session.playlist()` — get a specific playlist by ID
  - `spotify_session.current_user()` — get current user ID for ownership filtering

**Tidal API:**
- Purpose: Sync target — searches for tracks, creates/updates playlists and favorites
- SDK: `tidalapi ~= 0.8.10`
- Key calls (in `src/spotify_to_tidal/sync.py`, `src/spotify_to_tidal/tidalapi_patch.py`):
  - `tidal_session.search()` — search tracks and albums
  - `tidal_session.user.create_playlist()` — create new playlists
  - `tidal_session.user.favorites.add_track()` — add favorites
  - `tidal_session.playlist()` — get playlist by ID
  - Custom paginated wrappers in `src/spotify_to_tidal/tidalapi_patch.py` for bulk operations (fetching all playlists, tracks, favorites in parallel async chunks)

## Authentication

**Spotify — OAuth 2.0 (Authorization Code Flow):**
- Implementation: `src/spotify_to_tidal/auth.py` → `open_spotify_session()`
- Uses `spotipy.SpotifyOAuth` with:
  - `client_id` and `client_secret` from `config.yml`
  - `redirect_uri` from `config.yml` (default: `http://127.0.0.1:8888/callback`)
  - Scopes: `playlist-read-private`, `user-library-read`
- Token managed automatically by `spotipy` (file-based cache via spotipy internals)

**Tidal — OAuth 2.0 Device Flow:**
- Implementation: `src/spotify_to_tidal/auth.py` → `open_tidal_session()`
- First run: opens `login.verification_uri_complete` in browser via `webbrowser.open()`
- Subsequent runs: loads saved tokens from `.session.yml`
- Token persistence: reads/writes `token_type`, `access_token`, `refresh_token`, `session_id` to `.session.yml` via `yaml.dump`/`yaml.safe_load`
- Re-authenticates automatically if loading previous session fails

## Data Flow

```
config.yml (Spotify credentials)
    → SpotifyOAuth → spotipy.Spotify session
    → Fetch playlists + tracks (paginated, parallel async)
    → Track matching logic (ISRC, duration, name, artist fuzzy match)
    → Cache: TrackMatchCache (in-memory) + MatchFailureDatabase (SQLite .cache.db)
    → tidalapi.Session (OAuth device flow, token from .session.yml)
    → Create/update Tidal playlists and favorites
    → "songs not found.txt" (append-only log for unmatched tracks)
```

**Track Matching Strategy (in `src/spotify_to_tidal/sync.py`):**
1. ISRC exact match (preferred)
2. Duration match (±2 seconds) AND name match (substring) AND artist match (set intersection)
3. Two search modes: album-level search first, then standalone track search
4. Rate limiting: leaky-bucket semaphore (`max_concurrency` / `rate_limit` from config)

**Retry Behavior:**
- Up to 5 retries on `TooManyRequests`, `RequestException`, `SpotifyException`
- Exponential-ish backoff: 1s → 10s → 60s → 5min → 10min

## Configuration & Secrets

**Spotify credentials** are stored in `config.yml` (user-managed, not committed):
```yaml
spotify:
  client_id: <from Spotify Developer Dashboard>
  client_secret: <from Spotify Developer Dashboard>
  username: <Spotify username>
  redirect_uri: http://127.0.0.1:8888/callback
```

**Tidal tokens** are stored in `.session.yml` (auto-generated at runtime, not committed).

**`.gitignore` coverage:** Both `config.yml` and `.session.yml` should be excluded from version control. `example_config.yml` provides a safe template with placeholder values.

**No environment variables** are used — all credentials flow through `config.yml` and `.session.yml` files.

**Cache files:**
- `.cache.db` — SQLite, persists track match failures with exponential retry intervals
- `.cache-raphaelmue` — Spotipy's built-in token cache (named by username)

---

*Integration audit: 2026-03-30*
