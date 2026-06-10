# Design: `--map` GPS heatmap

## Goal

Add a `--map PATH` flag that renders a high-resolution PNG heatmap showing where
a set of photos were taken — a modern, Google-Photos-style visualization built
from the GPS EXIF already present in the photos. The mode is **read-only**: it
reads coordinates and renders; it never tags.

```bash
gpsphototag --photo ~/Pictures/Trip/ --map trip.png
```

## Look & feel

- **Basemap:** Carto "Positron" tiles — soft light-grey land, white water, faint
  labels (the clean, bright photo-app aesthetic).
- **Heatmap:** smooth density "glow" with a warm transparent→orange→red→white
  gradient. Low density is fully transparent so the basemap shows through.
- **Chart-junk free:** no axes, ticks, borders, or padding. Tight crop to the
  photo extent, anti-aliased, high DPI.

## Architecture

All changes are additive. Only `exif.py` (one new read function) and `cli.py`
(new flags + a `run_map` entry) touch existing files; the rest lives in a new
`mapper.py` module.

### 1. `exif.read_gps(path) -> tuple[float, float] | None`

The read-side mirror of the existing `has_gps()`. Returns `(lat, lon)` in signed
decimal degrees (WGS-84), or `None` when no usable GPS is present. Uses the same
dispatch as the rest of the module:

- JPEG / HEIC / PNG: Pillow GPS IFD — convert the DMS rationals plus the
  `GPSLatitudeRef`/`GPSLongitudeRef` (N/S/E/W) back into signed decimals.
- RAW: `raw_writer.read_raw_metadata` for embedded GPS, falling back to the XMP
  sidecar (consistent with how `has_gps` treats RAW).

### 2. `gpsphototag/mapper.py` (new module)

Owns all map-specific logic:

- `collect_coordinates(photos) -> (coords, n_with_gps, n_without)` — walk photos,
  call `read_gps`, return the coordinate list plus counts for reporting.
- `compute_extent(coords) -> (min_lon, min_lat, max_lon, max_lat)` — bounding box
  with a padding margin, and a **minimum span** so a single photo or a tight
  cluster still gets a sensible basemap zoom.
- `render_heatmap(coords, out_path, *, dpi)` — the rendering pipeline below.
- A guarded import of the optional deps. If they are missing, raise
  `MapDependencyError` carrying the "install the map extra" guidance.

### 3. Rendering pipeline (matplotlib + contextily + numpy)

1. Project coordinates to Web Mercator (the basemap's CRS).
2. Build a fine grid and accumulate a small 2-D Gaussian "splat" per photo onto
   it — a true KDE-style glow, implemented with numpy only (no scipy).
3. `contextily.add_basemap` fetches Carto Positron tiles for the extent.
4. Overlay the density grid with `imshow` using a custom
   transparent→orange→red→white `LinearSegmentedColormap` (alpha ramps from 0 at
   low density), blended over the basemap.
5. Strip axes/borders, set a tight layout, and `savefig` to PNG. Figure sized for
   a crisp ~2000px-wide image; `--map-dpi` (default 200) allows higher.

## CLI integration

- New flags:
  - `--map PATH` — output PNG; its presence activates map mode.
  - `--map-dpi N` — render DPI (default 200).
- Map mode is **read-only**. Combining `--map` with any writing flag (`--out`,
  `--overwrite`, `--replace`, `--gps`, `--maps-history`, `--fix-dates`) is a clear
  argparse error: *"--map is read-only; remove writing flags."*
- `validate_destination` is skipped in map mode (no `--out`/`--overwrite`
  requirement).
- A new `run_map(photos, out_path, dpi)` in `cli.py` keeps `main()` thin, mirroring
  the existing top-level structure. `main()` dispatches to it before the tagging
  path when `--map` is set.

## Error handling

| Condition | Behavior |
|---|---|
| Optional deps missing | `MapDependencyError` → "Map rendering requires the map extra: `pip install gpsphototag[map]`", exit 1 |
| No photo has GPS | Clear message including the with/without counts, exit 1 |
| Single point / tight cluster | `compute_extent` minimum-span fallback so the basemap zooms sanely |
| Tile fetch failure (offline) | Caught → "could not download map tiles — check your connection", exit 1 |

The logger reports `N of M photos had GPS` so the user knows what was plotted.

## Packaging

`pyproject.toml`:

- New optional extra: `map = ["matplotlib>=3.7", "contextily>=1.5"]` (numpy comes
  transitively via matplotlib).
- The same packages are added to the `dev` extra so the test suite can exercise
  rendering.

## Testing

Project coverage gate is 95% (`--cov-fail-under=95`).

- **`read_gps`:** round-trip a known lat/lon through `write_gps` then read it back
  (JPEG); RAW sidecar case; no-GPS → `None`.
- **`mapper`:** `collect_coordinates` counts; `compute_extent` padding and
  single-point minimum span (pure, deterministic); Gaussian-splat accumulation
  lands density in the expected grid cell.
- **`render_heatmap`:** monkeypatch `contextily.add_basemap` (no network) and
  assert a PNG of the expected pixel dimensions is written; simulate `ImportError`
  to cover the missing-deps path.
- **`cli`:** map-mode wiring, the writing-flag conflict error, and the
  no-GPS-found exit code.

Mock only externalities (the contextily tile fetch / network). Everything else —
EXIF read/write, grid math, CLI parsing — runs against the real code.

## Amendment 2026-06-10 — clustering, auto-zoom, modern glow

Refinements made during implementation:

- **Auto-zoom to fit all photos.** `render_heatmap` builds the window from the
  padded bounding box of the selected coordinates, projects to Web Mercator,
  fits it into an aspect range (widening the deficient axis only, never
  squashing), sizes the figure to match, uses `set_aspect("equal")`, and
  re-asserts the limits after the basemap + overlay are drawn. This keeps the
  view tight on the actual photos (a city stays a city, not all of Europe).
- **Multi-location clustering.** Photos are grouped into geographically distinct
  clusters via single-linkage on haversine distance (`cluster_coordinates`,
  threshold 50 km). When more than one cluster is found, the CLI lists them —
  reverse-geocoded to place names via `geopy` (`describe_location`, best-effort)
  — and resolves which to render from `--map-clusters` (`all` / `1,2`), an
  interactive prompt on a TTY, or "include all" non-interactively.
- **Modern semi-transparent palette.** The colormap is built from public RGBA
  stops (amber → orange → red → magenta) with alpha ramping 0 → ~0.72, so the
  basemap streets stay visible through a translucent glow.
- **Per-photo markers.** A crisp translucent dot is drawn at each photo on top
  of the glow, so individual locations stay visible regardless of cluster
  density (the glow alone, normalised to the busiest spot, hid sparse points
  like a few airport shots). Overlapping dots in busy areas darken naturally.
  Regression-tested via pixel inspection of an isolated point.
- **Packaging.** The `map` extra also includes `geopy` (used for cluster names).

New tested units: `haversine_km`, `cluster_coordinates`, `parse_cluster_selection`,
`describe_location` (geopy mocked), `_fit_aspect`, and the CLI cluster-selection
paths (flag, interactive, non-interactive, invalid).
