# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Tokyo Trip Companion — a personal, installable PWA trip planner (Tokyo-based trip, accommodation in Ebisu). See [README.md](README.md) for the full feature/tech overview; this file focuses on conventions and architecture a fresh session needs.

## Development

No build step, no package manager, no test suite. Everything lives in `index.html` (HTML + CSS + JS in one file, rendered via manual `innerHTML` updates — no framework, no virtual DOM). To work on it, just edit `index.html` and open it in a browser (or serve the folder statically); reload to see changes. There is nothing to build or lint.

Deployment is via GitHub Pages, serving directly from the `main` branch root — pushing to `main` redeploys automatically, usually within ~1 minute.

**The deployed URL/origin must never change.** All user data lives in `localStorage`, which is bound to the exact origin+path. Changing the repo name, custom domain, or hosting path would silently orphan every user's saved trip data.

## State & rendering model

- Single global `state` object (shape documented in README, constructed by `defaultState()`), persisted via `storageAdapter` (uses `window.storage` if present — i.e. running as a Claude Artifact — else falls back to `localStorage` under key `trip-state`).
- `mutate(fn)` — applies `fn(state)`, schedules a debounced save, and calls `render()` (full re-render of the active tab via `innerHTML`).
- `mutateSilent(fn)` — same but skips `render()`. Used for text inputs on every keystroke so the cursor doesn't jump on re-render. These inputs pair `input` (silent) with a `change` listener that calls `render()` once the field loses focus, so downstream derived UI (e.g. the trip timeline, the countdown) picks up the edit without disrupting typing.
- `render()` dispatches on `state.activeTab` and calls the matching `render*()` function, then `attachTabEvents()` to rewire listeners on the fresh DOM (everything is torn down and rebuilt on every render — there's no diffing).

## The privacy boundary: `defaultState()` must stay empty

`defaultState()` intentionally returns **empty/generic structures only** (no places, no neighborhoods, no checklist items, no overview text). This is deliberate: the git repo should contain only the reusable app shell, never personal trip content. Do not reintroduce hardcoded trip-specific data into `defaultState()` or anywhere else in the template (there was previously a hardcoded countdown target date and a hardcoded "Check-in Ebisu" label baked into the render template itself, outside of `state` — both were bugs precisely because they leaked personal content into the generic app and were disconnected from what the user could actually edit).

The user's real trip data lives only in their browser's `localStorage` and in a local, gitignored backup file (`current-trip-data-export.json` — real content, never commit it). It's restored on a fresh install via the manual export/import feature (bottom of the **Übersicht** tab, not a separate "Reise" tab), which round-trips the entire `state` object as JSON.

## Map subsystem

Three interchangeable map engines with a runtime fallback chain, orchestrated by `initMap()`: **Google Maps** (if `state.settings.googleMapsKey` set) → **MapTiler** (if `maptilerKey` set) → **Leaflet + OpenStreetMap** (no key, always works). Each engine has its own `init*Map()` function and its own marker API, but shares engine-independent helpers:

- `markerStyle(nb, activeNbIds)` — styling for **neighborhood** dots (all `state.neighborhoods` are always plotted, at small/dim size if they have no places, larger/colored if they do or are actively filtered).
- `filteredPlaces()` — the currently filtered `state.places` list, shared by both the Orte-tab list and the map's individual place pins.

**Neighborhoods vs. places** are two distinct concepts, don't conflate them: neighborhoods are a curated, informal list of districts/day-trip destinations (not official administrative boundaries) always shown on the map as reference dots. Places optionally get their own precise pin if geocoded (`address`/`lat`/`lng` fields) — via `geocodeQuery()` (Nominatim forward search, also accepts pasted `lat, lng` directly since Nominatim frequently fails on precise romanized Japanese addresses) and `resolveNeighborhoodForCoords()` (snaps to an existing neighborhood within 5km, otherwise reverse-geocodes and creates a new one — because Nominatim's own administrative boundaries rarely match this app's informal neighborhood granularity).

The "Stadtteile" filter chips only list neighborhoods that have at least one place assigned (a filter chip for an empty neighborhood would do nothing) — this is why the map can show more dots than the filter chip list; that's expected, not a bug.
