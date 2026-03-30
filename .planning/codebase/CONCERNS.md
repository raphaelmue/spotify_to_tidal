# Technical Concerns

**Analysis Date:** 2026-03-30

## Known Issues

**Bare `assert` used for runtime control flow:**
- Location: `src/spotify_to_tidal/sync.py:111`
- Code: `assert( not len(album_tracks) == album.num_tracks ) # incorrect metadata :(`
- Impact: Running Python with `-O` (optimizations) strips asserts; this check silently disappears. Also signals an acknowledged third-party metadata bug with no real recovery path — the code just continues to the next album.

**`sys.exit(1)` inside async retry logic:**
- Location: `src/spotify_to_tidal/sync.py:155`
- Impact: `repeat_on_request_error` calls `sys.exit(1)` after exhausting retries. This is abrupt, leaves no cleanup opportunity, and cannot be caught by callers. A raised exception would be far safer.

**Format string bug in error message:**
- Location: `src/spotify_to_tidal/auth.py:27`
- Code: `sys.exit("Error opening Spotify sesion; could not get token for username: ".format(config['username']))`
- Impact: The `.format()` call has no `{}` placeholder, so the username is never inserted into the error message. Typo in "sesion" also present.

**"songs not found.txt" written to CWD, not configurable:**
- Location: `src/spotify_to_tidal/sync.py:283-288`
- Impact: Output file path is hardcoded as `"songs not found.txt"`. File is opened in append mode on every run, so it accumulates across runs with no rotation or clearing. Location cannot be controlled via config.

**`import math` duplicated:**
- Location: `src/spotify_to_tidal/sync.py:9` and `sync.py:20`
- Impact: Minor — duplicate import is dead weight, but no runtime effect.

## Technical Debt

**Global mutable singleton caches in module scope:**
- Location: `src/spotify_to_tidal/cache.py:83-84`
- Code: `failure_cache = MatchFailureDatabase()` and `track_match_cache = TrackMatchCache()`
- Impact: Module-level singletons are instantiated on import. `TrackMatchCache.data` is a class-level (not instance-level) dict, meaning all instances share the same dictionary. This makes isolated testing difficult and can cause cross-test contamination.

**`tidalapi_patch.py` accesses private API internals:**
- Location: `src/spotify_to_tidal/tidalapi_patch.py`
- Accesses: `playlist._etag`, `playlist._base_url`, `playlist._reparse()`, `session.request.map_request()`, `session.request.map_json()`, `user.playlist.parse_factory`
- Impact: Any minor tidalapi version bump can silently break the entire sync. This is a significant fragility point given the `tidalapi~=0.8.10` pin allows patch updates.

**`_get_all_chunks` mutates params dict in place:**
- Location: `src/spotify_to_tidal/tidalapi_patch.py:36-38`
- Code: `new_params = params` followed by `new_params['offset'] = offset`
- Impact: This is not a copy — it modifies the caller's dict. The `offset` key is added to the original `params` dict on every call.

**Rate limiter implemented as a custom leaky-bucket using a semaphore:**
- Location: `src/spotify_to_tidal/sync.py:250-260`
- Impact: Non-standard, hard to tune correctly. The `semaphore.release()` calls inside a list comprehension for side effects are idiomatic Python abuse. `[semaphore.release() for i in range(new_items)]` should be a loop.

**Playlist matching by name only:**
- Location: `src/spotify_to_tidal/sync.py:364-368`
- Impact: If two Spotify playlists have the same name, or a Tidal playlist name collision exists, the wrong playlist may be matched. No ID-based fallback exists for the user-playlist flow.

## Security Concerns

**Tidal OAuth tokens written to `.session.yml` in plaintext:**
- Location: `src/spotify_to_tidal/auth.py:58-62`
- Impact: `access_token` and `refresh_token` are written to a YAML file on disk. The file is `.gitignore`d but still poses a local security risk if the working directory is shared or world-readable.

**`config.yml` contains Spotify `client_secret` in plaintext:**
- Location: Referenced in `src/spotify_to_tidal/type/config.py` (`SpotifyConfig.client_secret`)
- Impact: The config file pattern stores API credentials in plain YAML. No environment variable or secrets manager fallback is supported by the config schema.

**No input sanitization on playlist names used in file output:**
- Location: `src/spotify_to_tidal/sync.py:284-288`
- Impact: Playlist names from Spotify are written verbatim to `"songs not found.txt"`. Pathological playlist names are unlikely to cause directory traversal here, but no validation occurs.

## Performance Concerns

**`populate_track_match_cache` is O(n*m) nested loop:**
- Location: `src/spotify_to_tidal/sync.py:194-219`
- Impact: For large playlists (e.g., 1000+ tracks), this performs up to n*m match comparisons. Each comparison itself calls `isrc_match`, `duration_match`, `name_match`, and `artist_match`. No indexing or early-exit optimization exists beyond popping matched items.

**`add_multiple_tracks_to_playlist` chunk indexing off-by-one risk:**
- Location: `src/spotify_to_tidal/tidalapi_patch.py:26`
- Code: `playlist.add(track_ids[offset:offset+chunk_size])` uses `chunk_size` rather than `count`
- Impact: When `count < chunk_size` (final chunk), this still slices `offset:offset+chunk_size` which Python handles safely, but `count` is computed and then ignored, making the logic misleading.

**`clear_tidal_playlist` is synchronous and sequential:**
- Location: `src/spotify_to_tidal/tidalapi_patch.py:14-19`
- Impact: Deletes tracks in sequential 20-item chunks. For large playlists this is slow because `_reparse()` is called after each batch and each DELETE is a network request. No async or batching optimization.

## Dependency Risks

**`tidalapi~=0.8.10` pin allows patch updates that can break private API usage:**
- The patch relies on `_etag`, `_base_url`, `_reparse`, `map_request`, `map_json`, and `parse_factory` internals.
- A patch release of `tidalapi` could remove or rename any of these without a semver major bump.

**`pytest` and `pytest-mock` listed as runtime dependencies in `pyproject.toml`:**
- Location: `pyproject.toml:16-17`
- Impact: Test frameworks are in `[project.dependencies]`, not a `[project.optional-dependencies]` dev group. Users installing the package via pip get test tools installed into their environment.

**CircleCI config uses deprecated `circleci/python:3.10` convenience image:**
- Location: `.circleci/config.yml:6,17`
- Impact: `circleci/python` images are deprecated in favor of `cimg/python`. The CI environment may drift from supported images.

**`requests_timeout=2` is very tight:**
- Location: `src/spotify_to_tidal/auth.py:22`
- Impact: A 2-second timeout on the Spotify OAuth token exchange will fail on slow network connections, causing a misleading `SpotifyOauthError`.

## Missing Features / Gaps

**No dry-run mode:** No way to preview what changes would be made without actually writing to Tidal.

**No progress persistence between partial runs:** If a sync of 500 playlists fails at playlist 400, the next run restarts from scratch. The failure cache only tracks individual track matches, not playlist-level progress.

**No support for Spotify podcast episodes in playlist output:** Episodes filtered out (`type == 'track'` check) but no user-visible summary of how many were skipped.

**No logging framework:** All output is via raw `print()` calls. There is no log level control, no log file, and no structured logging. Debugging requires reading stdout.

**`TrackMatchCache` has no persistence:** The in-memory `TrackMatchCache` (Spotify ID → Tidal ID mappings) is lost on every run. Successful matches are re-discovered on each sync, wasting Tidal API quota.

## Maintenance Concerns

**`sync.py` is a 419-line monolithic module:**
- Contains matching logic, rate limiting, playlist sync, favorites sync, Spotify fetch, and Tidal write operations all mixed together.
- Adding new sync targets or matching strategies requires editing this single file.

**No type annotations on many internal functions:**
- `normalize`, `simple`, `tidal_search`, `_run_rate_limiter`, `get_new_tidal_favorites` lack full type signatures.
- Makes refactoring harder and IDE support weaker.

**Test coverage is minimal:**
- Only `auth.py` and `cache.py` have tests.
- `sync.py`, `tidalapi_patch.py`, and `__main__.py` have zero test coverage.
- The most complex matching logic (`match`, `name_match`, `artist_match`, `tidal_search`) is completely untested.

## Opportunities

**Add `TrackMatchCache` persistence to SQLite:** The existing `MatchFailureDatabase` already uses SQLAlchemy + SQLite. Adding a second table for successful matches would eliminate redundant Tidal searches on re-runs.

**Extract matching logic into a dedicated module:** Functions `isrc_match`, `duration_match`, `name_match`, `artist_match`, `match` are pure and side-effect-free — ideal candidates for a `matching.py` module with comprehensive unit tests.

**Replace `print()` with Python `logging` module:** Single-line change to add log levels, file output, and structured logging without breaking existing behavior.

**Fix `params` mutation in `_get_all_chunks`:** Replace `new_params = params` with `new_params = {**params}` to avoid mutating the caller's dict.

**Move test dependencies to optional dev extras in `pyproject.toml`:** Add `[project.optional-dependencies] dev = ["pytest~=8.0", "pytest-mock~=3.8"]` and remove from runtime deps.

---

*Concerns audit: 2026-03-30*
