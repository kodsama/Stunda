# GPSPhotoTag 2.0 — Flutter desktop + headless Dart CLI

**Status:** approved 2026-06-24 · **Supersedes:** the Python implementation (kept as reference until parity is verified).

## 1. Goal & scope

Rebuild GPSPhotoTag as a cross-platform Flutter **desktop** app (macOS/Linux/Windows)
plus a headless, **LLM-friendly** Dart CLI, both powered by the *same* Flutter-free
engine. Pure Dart — no Python runtime.

**In scope (parity with the Python app):**

- `tag` — write GPS EXIF from a GPX track or Google location history (GPX > Google,
  exact or linearly interpolated, `maxTimeDiff` threshold, timezone handling).
- `map` — render a density heatmap PNG (auto-zoom, per-cluster zooms, filename-range
  labels), read-only.
- `prune-raw` — move RAW files lacking a same-name JPG/HEIC companion (tree-wide) to
  Trash, or delete with `--rm`.
- `fix-dates` — `exif` (file date ← EXIF) or `file` (EXIF ← file date).
- Formats: JPEG/JPEG (lossless inline), PNG (inline), RAW (XMP sidecar default or
  exiftool embed), HEIC (via libheif).

**Dropped from the original template prompt (do not build):** ML model inference,
model/engine/language catalog, "install all languages", generic `convert`.

**Reinterpreted template ideas (build these):**

- "Dependency/toolkit check + Install buttons" → checks **exiftool** (RAW embed +
  Fuji/Canon timestamp read) and **libheif** (HEIC), reachable tile endpoint;
  auto-install via Homebrew/apt/winget.
- "Pluggable backends (local + cloud)" → **location sources** (GPX, Google
  Takeout/Timeline), **geocoder** providers, **tile** providers.
- "Downloadable asset catalog" → basemap-tile / geocoder providers (size,
  attribution, Recommended badge, retry) — **not** ML models.

## 2. Architecture — strict layers, Flutter-free engine

Pub workspace (Dart 3 `resolution: workspace`):

```
packages/engine/   gpsphototag_engine — pure Dart, NEVER imports flutter or dart:ui
  domain/    immutable models + enums (TimedPoint, PhotoRow, LocationResult,
             Status, TagOptions, MapOptions, EngineEvent)
  data/      exif backends, gpx/google parsers, geocoder, tile fetcher,
             process runner, file system, clock — all behind interfaces
  services/  locator, tagger, pruner, dater, mapper, toolkit_check
  app/       orchestrators that compose services and yield Stream<EngineEvent>
packages/cli/      gpsphototag_cli — depends on engine only
app/               gpsphototag_gui — Flutter app; depends on engine only
```

The engine is the single source of truth. CLI and GUI are thin adapters over the
same `app/` orchestrators. **Every external seam is an injectable interface** with a
real and a fake implementation: `ProcessRunner`, `HttpClient`, `ExifBackend`,
`LocationSource`, `Geocoder`, `TileProvider`, `Clock`, `FileSystemPort`.

Functions are small and composable; big operations compose smaller ones with explicit
options objects. Dartdoc on every public API; inline comments only for non-obvious
logic.

## 3. Data flow

`Input → collect paths → parse sources → for each photo: read timestamp → locator
(GPX>Google, exact|interpolated, threshold) → writer (format dispatch) → EngineEvent`.
The orchestrator yields a typed event per item; the CLI serialises events to JSON
lines, the GUI feeds them to controllers.

## 4. Concurrency (critical)

Heavy/blocking work (EXIF I/O, image re-encode, Takeout zip parse, tile compositing)
runs in **worker isolates** (`Isolate.run` + a bounded pool). Events return over a
`SendPort`, re-broadcast as a `Stream`. The UI isolate only renders. A cancellation
token threads through the pool. The UI must stay responsive with live progress.

## 5. CLI + LLM contract

Commands: `info`, `list-sources`, `list-providers`, `check`, `tag`, `map`,
`prune-raw`, `fix-dates`, `schema`.

- `schema` emits JSON describing every command, option, event type, and exit code for
  agent discovery.
- `--json` emits one JSON object per line on stdout:
  `{"event":"log|progress|item|done|error", ...}`. Human mode prints a table/summary.
- Exit codes: `0` ok · `2` completed with no-match/partial · `3` bad input · `4`
  missing toolkit · `5` internal error.
- Ship a compiled standalone binary (`dart compile exe`).
- `AGENTS.md` documents the discovery flow (`schema` → pick command → run `--json`)
  and the event contract.

## 6. GUI / UX

**Stepped, collapsible walkthrough** — one step expanded at a time, auto-advancing;
completed steps collapse with a ✓ and stay tappable; the active step's **Continue** is
enabled only when satisfied:

1. **Toolkit check** (runs first; blocks until env is genuinely usable)
2. **Pick input** (native picker)
3. **Review parsed input** (metadata summary + per-item include/exclude checklist)
4. **Options** (all exposed, smart defaults pre-selected)
5. **Output** (filename + destination folder, optional asset/source override)
6. **Run** (global progress bar + per-item live status; errors surfaced inline)
7. **Result** (summary + map preview)

- **Toolkit checker:** per-item status + **Install** button (Homebrew/apt/winget);
  never reports "Ready" unless usable; disables dependent options (e.g. RAW-embed
  needs exiftool) with an inline reason.
- **Errors in the UI**, not just logs. **Floating activity-log button** (bottom-right)
  with unread badge → slide-over panel; click-outside closes.
- **Theme:** distinctive cartographic identity — warm "field-notebook" palette
  (ink/paper/terracotta + topographic-contour accent), real type scale (display face
  for headers, humanist sans for body, tabular figures for coordinates). Not
  Material-default.
- **Logo:** hand-authored **SVG** (map pin merging with a photo corner / aperture),
  wired as app icon (all sizes, all 3 OSes) + in-app header.

## 7. Map rendering

Heatmap lives in the **engine** (Flutter-free) so CLI and isolates can produce it:
fetch tiles (`http`) → composite with the pure-Dart `image` package → density blobs +
per-photo dots → write PNG; auto-zoom, per-cluster outputs, collapsed filename-range
labels. The GUI displays the produced PNG. Interactive in-GUI map (`flutter_map`) is a
UI-layer stretch goal, never blocking.

## 8. Backends / providers (local-first, free by default)

- **Location sources:** GPX; Google Takeout `Records.json`; Google Timeline JSON/KML
  (auto-detected). Takeout `.zip` extraction in an isolate.
- **Geocoders:** OSM Nominatim (default, free) behind `Geocoder`; cloud key-based
  optional.
- **Tile providers:** CARTO/OSM no-key (default) behind `TileProvider`; others
  optional. Catalog shows size/attribution/Recommended + retry-on-failure.

## 9. Testing & process

- **Unit** (engine, with fakes), **widget** (each walkthrough step), **integration**
  (real conversions on generated fixtures, mirroring the Python `conftest`).
- Analyzer clean (`dart analyze`), `dart format`. Commit frequently.
- **Verify on real builds** — actually run `tag`/`map`/`prune` and confirm outputs;
  report honestly when upstream (exiftool/libheif/tiles) is unavailable instead of
  faking success.
- Detailed **README** + **ARCHITECTURE** docs.

## 10. Milestones

1. Workspace scaffold + engine skeleton + domain models + analyzer/CI + first tests.
2. Engine services: parsers, locator, writers (JPEG/PNG/sidecar + exiftool/libheif
   backends), pruner, dater, toolkit-check — full unit coverage.
3. CLI: all verbs + `--json` + `schema` + AGENTS.md + compiled binary; verified on
   real photos.
4. Map renderer in engine (PNG, auto-zoom, labels).
5. Flutter GUI: theme + logo + walkthrough + toolkit checker + picker + options +
   progress + activity log; isolate wiring.
6. Integration/widget tests, docs, packaging/icons for all three OSes.

## 11. Feasibility notes (honest)

- HEIC decode and RAF/CR3 **embed** require external tools (libheif/exiftool),
  surfaced + installable via the checker; the pure-Dart path always offers JPEG/PNG
  inline + RAW **sidecars**.
- exiftool on Windows needs its bundled Perl exe — checker detects/installs via winget,
  degrades gracefully if absent.
- Interactive in-GUI map is a stretch goal; the guaranteed deliverable is the
  engine-rendered PNG shown in the GUI.
