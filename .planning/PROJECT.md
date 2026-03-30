# spotify-to-tidal Desktop App

## What This Is

A cross-platform desktop application that wraps the existing spotify-to-tidal Python sync tool in a beautiful, easy-to-use interface. Users connect their Spotify and Tidal accounts, pick which playlists to sync, and hit a button — no terminal required. The app replaces the CLI as the primary interface and ships as a lightweight, installable binary for Windows, macOS, and Linux.

## Core Value

Anyone can sync their Spotify playlists to Tidal without touching a terminal or config file.

## Requirements

### Validated

(None yet — ship to validate)

### Active

- [ ] User can connect their Spotify account via OAuth
- [ ] User can connect their Tidal account via OAuth
- [ ] User can browse and select which Spotify playlists to sync
- [ ] User can trigger a sync and see real-time progress
- [ ] App ships as a standalone installable binary (no Python required)
- [ ] App works on Windows, macOS, and Linux
- [ ] Existing sync logic bugs are fixed (dict mutation, class-level cache, tidalapi fragility)
- [ ] Core sync logic has meaningful test coverage

### Out of Scope

- Syncing from Tidal → Spotify (reverse direction) — not in scope for v1
- Mobile app — desktop only
- Hosted/cloud sync — local app only, credentials stay on user's machine
- Rewriting sync logic in Rust/JS — Python sidecar keeps existing logic intact

## Context

- Existing codebase: Python CLI tool using `spotipy` (Spotify) and `tidalapi` (Tidal)
- Key bugs surfaced by codebase map: `params` dict mutation in `_get_all_chunks`, class-level `TrackMatchCache.data` shared across instances, heavy reliance on tidalapi private internals (`_etag`, `_base_url`, `_reparse`)
- Zero test coverage on `sync.py` and `tidalapi_patch.py` — the most critical files
- Tech stack decision: **Tauri** (Rust native shell + webview) + React/Svelte frontend + Python sidecar for sync logic
  - Rationale: ~5MB installer vs ~150MB Electron, uses OS webview, web-native UX, keeps Python sync code intact

## Constraints

- **Distribution**: Must be installable by non-technical users — no Python, no pip, no CLI setup required
- **Size**: Lightweight — target <20MB installer
- **Privacy**: OAuth credentials and tokens stay local (no server, no cloud)
- **Compatibility**: Windows 10+, macOS 12+, Ubuntu 20.04+
- **Backwards compat**: CLI must still work for existing users

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Tauri over Electron | Lighter weight, smaller bundle, better for distribution | — Pending |
| Python sidecar over rewrite | Keeps existing sync logic, avoids rewriting Spotify/Tidal API integrations | — Pending |
| Fix bugs alongside UI | Reliability improvements make the UI experience trustworthy | — Pending |

---
*Last updated: 2026-03-30 after initial questioning*
