# Testing

**Analysis Date:** 2026-03-30

## Test Framework

**Runner:** pytest ~8.0
- Config: `pytest.ini`
- Options: `--maxfail=1 --disable-warnings`
- Test discovery: `testpaths = tests`, files matching `test_*.py`, classes `Test*`, functions `test_*`

**Mocking:** pytest-mock ~3.8 (provides `mocker` fixture wrapping `unittest.mock`)

**Run Commands:**
```bash
# Run all tests
pytest

# Run with verbose output
pytest -v

# Run specific test file
pytest tests/unit/test_cache.py

# Run specific test function
pytest tests/unit/test_auth.py::test_open_spotify_session
```

No coverage configuration detected (no `pytest-cov` in `pyproject.toml`, no `[coverage]` section).

## Test Structure

**Location:** `tests/` directory, separate from `src/`

**Directory layout:**
```
tests/
├── conftest.py          # Path setup: inserts src/ into sys.path
└── unit/
    ├── __init__.py
    ├── test_auth.py     # Tests for src/spotify_to_tidal/auth.py
    └── test_cache.py    # Tests for src/spotify_to_tidal/cache.py
```

**Naming:**
- Test files: `test_<module>.py` mirroring source module name
- Test functions: `test_<function_name>` or `test_<function_name>_<scenario>`

## Test Types

**Unit tests only** — no integration or end-to-end tests detected.

Covered modules:
- `auth.py` — `tests/unit/test_auth.py` (2 tests)
- `cache.py` — `tests/unit/test_cache.py` (5 tests)

**Not covered:**
- `sync.py` — no tests (largest and most complex module)
- `tidalapi_patch.py` — no tests
- `__main__.py` — no tests
- `type/spotify.py` — no tests (TypedDict definitions, lower risk)

## Coverage

**Approximate coverage:** Low. Only 2 of 6 source modules have any tests. The core business logic in `sync.py` (~419 lines) has zero test coverage.

**What is tested:**
- Spotify OAuth session creation (happy path and OAuth error path)
- `MatchFailureDatabase`: insert, query, and delete operations via in-memory SQLite
- `TrackMatchCache`: insert and get operations

**What is not tested:**
- All track matching logic (`isrc_match`, `duration_match`, `name_match`, `artist_match`, `match`) in `sync.py`
- `tidal_search` async search logic
- `repeat_on_request_error` retry behavior
- Playlist sync orchestration (`sync_playlist`, `sync_favorites`)
- Tidal session opening (`open_tidal_session`) — relies on real OAuth flow
- Pagination helpers in `tidalapi_patch.py`

## Mocking Strategy

**Tool:** `pytest-mock` `mocker` fixture (wraps `unittest.mock.patch`)

**Pattern — patch at the point of use:**
```python
# Patch where the name is imported, not where it's defined
mock_spotify_oauth = mocker.patch(
    "spotify_to_tidal.auth.spotipy.SpotifyOAuth", autospec=True
)
```

**`autospec=True` is used** on external library classes to ensure mock signatures match real APIs.

**In-memory database fixture for SQLite:**
```python
@pytest.fixture
def in_memory_db():
    engine = create_engine("sqlite:///:memory:")
    return engine

def test_cache_match_failure(in_memory_db, mocker):
    mocker.patch(
        "spotify_to_tidal.cache.sqlalchemy.create_engine", return_value=in_memory_db
    )
    failure_db = MatchFailureDatabase()
    ...
```

**What is mocked:**
- External SDK constructors (`spotipy.SpotifyOAuth`, `spotipy.Spotify`)
- Database engine creation (`sqlalchemy.create_engine`) — replaced with in-memory SQLite
- `sys.exit` — patched to prevent test process termination during error-path tests

**What is NOT mocked:**
- `MatchFailureDatabase` and `TrackMatchCache` are tested against real (in-memory) implementations, not mocked

## Test Data

No shared fixtures file beyond `conftest.py` (which only handles `sys.path`). Test data is defined inline as dicts within each test function:

```python
mock_config = {
    "username": "test_user",
    "client_id": "test_client_id",
    "client_secret": "test_client_secret",
    "redirect_uri": "http://127.0.0.1/",
    "open_browser": True,
}
```

No factory libraries (e.g., `factory_boy`) or shared fixtures for Spotify/Tidal track objects.

## Adding New Tests

- Place new unit test files under `tests/unit/test_<module>.py`
- Import source modules directly: `from spotify_to_tidal.sync import match, name_match`
- Use `mocker` fixture (from `pytest-mock`) for patching external dependencies
- For async functions, use `pytest-asyncio` (not currently installed — would need adding to `pyproject.toml`)

---

*Testing analysis: 2026-03-30*
