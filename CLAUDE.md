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

- `markerStyle(nb, activeNbIds)` — styling for **neighborhood** dots. All `state.neighborhoods` are always plotted; larger/colored if they have places or are actively filtered, small/dim otherwise. Whenever any Stadtteil filter is active, every *non*-selected, non-home neighborhood additionally gets forced into a `dimmed` (tiny, no label) state so the selected one doesn't compete with ~30 other dots at low zoom — check `st.dimmed` before drawing a permanent label in any engine.
- `filteredPlaces()` — the currently filtered `state.places` list, shared by the Orte-tab list and the map's place pins.
- `placeMarkerPosition(p)` — **only returns non-null while `state.filters.nb` includes that place's neighborhood** (place pins are intentionally hidden on the unfiltered, zoomed-out map to avoid clutter). Returns the real `lat`/`lng` if geocoded, otherwise a small deterministic jitter around the neighborhood's own dot (`approx:true`) so filtering by Stadtteil shows every place, not just the geocoded ones. Every engine renders `approx` pins smaller/translucent vs. the solid pin for exact positions — don't leave `icon: undefined` for exact places, that silently falls back to Google's stock red marker instead of the intended custom dot (this exact bug shipped once already).

**Neighborhoods vs. places** are two distinct concepts, don't conflate them: neighborhoods are a curated, informal list of districts/day-trip destinations (not official administrative boundaries) always shown on the map as reference dots. Places optionally get their own precise pin if geocoded (`address`/`lat`/`lng` fields).

### Geocoding & neighborhood auto-assignment

- `geocodeQuery(query)` — single-result Nominatim forward search, used for the "Neuer Stadtteil" quick-add and the Stadtteile-manager's search.
- `geocodeSearch(query)` — multi-result (up to 5) Nominatim search rendered as a picker in the place form, closer to a Google-Maps-style search box than blindly trusting the first hit. Both accept pasted `lat, lng` directly as a bypass, since Nominatim frequently fails on precise romanized Japanese addresses (chōme/banchi-level detail, or even uncommon romanized district names resolve to zero results while the plain neighborhood name works).
- `resolveNeighborhoodForCoords(lat, lng)` — snaps to an existing neighborhood within 5km (`NEARBY_NEIGHBORHOOD_KM`); otherwise reverse-geocodes the point to get an area name and **forward-geocodes that name** to get a proper centroid before creating a new neighborhood — never reuse the raw queried point as the new neighborhood's coordinates, since the resolved name is often a broad existing area (e.g. an offshore beach reverse-geocoding to "Fujisawa" city must not plant the whole "Fujisawa" dot on that beach). Auto-created neighborhoods can still end up misplaced (Nominatim's centroid choice isn't always the visual center) — that's what the Stadtteile-manager (below) is for.
- Clicking an empty spot on the map (all three engines) opens the new-place form via `openPlaceForm(null, {lat, lng[, name]})`, pre-filling the location and triggering the same neighborhood auto-assignment as a manual search pick.
- Google Maps only: clicking a POI resolves its name via `PlacesService.getDetails()` (requires the **separate, billed Places API** enabled — enabling only "Maps JavaScript API" is not enough, calls fail with `REQUEST_DENIED` otherwise) and shows a fixed **"+ Hinzufügen"** button pinned to the bottom of the map card (`getOrCreatePoiAddButton()`). Google's own default POI info window is a closed UI element that can't be customized or injected into directly — don't try to layer a custom `InfoWindow` at the same anchor point, it renders completely hidden behind Google's own (this shipped once already); a fixed-position button outside the map's coordinate space is the only reliable option that survives the info window's own sizing/anchoring.

### Stadtteile manager

`openNeighborhoodManager()` (a "Verwalten" link next to the Stadtteile section header) lists every neighborhood with editable name/lat/lng and a 🔍 search button (reuses `geocodeSearch()` against the row's current name) plus delete (warns if places are still assigned). This exists because auto-created neighborhoods have no other way to be corrected or removed short of hand-editing the export/import JSON.

The "Stadtteile" filter chips only list neighborhoods that have at least one place assigned (a filter chip for an empty neighborhood would do nothing) — this is why the map can show more dots than the filter chip list; that's expected, not a bug.

## Cross-device sync (GitHub Gist)

Optional, opt-in via `state.settings.githubToken` (a GitHub PAT, scope `gist`) and `state.settings.gistId`. `scheduleSave()` — already debouncing the `localStorage` write — also debounce-triggers `scheduleGistPush()` → `pushToGist()`, and `loadState()` does a silent `pullFromGist()` on startup if both fields are set. There is **no real-time sync and no merge**: it's push-on-change / pull-on-load only, and a pull is a wholesale `Object.assign(state, remote)` — whichever device syncs last simply overwrites the other's data. `gistId` is deliberately a manually-editable field (not just internal state) so a second device can be pointed at the first device's already-created gist instead of each device silently creating its own separate one. Token/gist-id inputs strip *all* whitespace (not just `.trim()`), since a token pasted via a notes app/messenger can pick up an embedded line break mid-string that a plain trim won't catch.
