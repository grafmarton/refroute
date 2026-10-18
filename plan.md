# Plan: Human-in-the-Loop Reference Route Tool

Status: implemented and accepted; reworked from v1 "chained legs" to
segment-cached incremental routing (no legs), acceptance PASS.

---

## 1. Project Context (why this tool exists)

Downstream goal (out of this tool's scope, but dictates its contract):

Vehicle GPS traces (~50k points, ~300 km per drive, mountainous terrain with
tunnels/valleys causing drift and signal drops) must be labeled with:

- **OSM road type** (highway tag)
- **Structure flags** (tunnel / bridge) — needed because external elevation data
  returns ground surface there, not road height; gaps closed by interpolation
- **Elevation/slope** sampled along the *ideal road mid-line*, not the noisy GPS
  measurement (speed comes from vehicle CAN, not GPS)

Labels → segmentation into homogeneous stretches (e.g. continuous primary road)
→ pooling of segments into a **load collective** approximating a target mix of
distance, road types, speed and slope distributions.

Automated map-matching was tried with mixed results: it usually fixes easy
sections but always fails somewhere (tunnels, drift onto parallel/wrong roads,
off-road artifacts). The dataset is small enough that a human in the loop is
viable — and gives per-metre control over artifacts no algorithm handles well.

**This project's scope: produce the idealized reference route only.**
Everything after the export (projection, interpolation, Overpass/PBF attribute
joins, elevation, segmentation, load collective) happens separately in Python.

---

## 2. Core Concept

A standalone browser tool where the raw GPS trace is only a visual **ghost
layer**, and a handful of user-placed waypoints generate a clean, topologically
correct route through the road network via the public OSRM API. The user
inspects the result visually and exports the merged route geometry for use as
ground truth in the notebook.

---

## 3. Locked Decisions

### 3.1 Scope

**In**
- Load trace CSV → ghost line on map
- Waypoint-driven routing (add / drag / reorder / line-drag via insertion)
- Incremental segment routing: consecutive waypoint pairs routed independently,
  cached, re-used across edits (no chained-leg concept)
- Snap-and-freeze: after routing, each waypoint is replaced by the exact
  geometry vertex on the road → all segments share vertices by construction
- Save/load full state (waypoints + segments) as JSON v2
- Visual verification (eyeball)
- Export: one merged dense route line as GeoJSON

**Out / deferred**
- Road-class, tunnel/bridge labels → later Python step (Overpass or local PBF)
- Elevation, slope, segmentation, load collectives → later Python step
- **OSM way IDs in the export → v2** (public OSRM returns no way IDs anywhere;
  verified live: response contains geometry, name/ref, node IDs via
  `annotations=nodes`, but no `osm_id`/way IDs). v1 Python side does a trivial
  nearest-way join against the already-clean geometry.
- Free-hand off-network segments (parking lots etc.): rare, undecided — schema
  may later add segment types `{osrm, manual}`, not built now.
- Deviation-stats / heat-map verification: not built (visual eyeball chosen).

### 3.2 Workflow

1. Notebook exports trace as **CSV** with ordered `latitude,longitude` columns.
2. Open `index.html` directly via `file://` — **no local HTTP server needed**
   (no CORS issue: file input + Blob download both work from `file://`).
3. CSV loads as a faint ghost line; map fits bounds. Loader is tolerant:
   auto-detect delimiter (`,` `;` tab), header names (case-insensitive
   `latitude|lat`, `longitude|lon|lng`), decimal-comma values; GeoJSON
   (LineString/MultiLineString/FeatureCollection) accepted as a fallback input.
   One trace per session; loading a new file replaces it.
4. Routing interaction:
   - 1st waypoint = start, nothing routes yet
   - 2nd waypoint = destination → route traced
   - 3rd+ waypoints refine the route (LRM re-routes on change)
   - Drag the route line itself to insert a via (LRM `linetouched` →
     `dragNewWaypoint`) — verified present in LRM source
   - Drag existing markers to move them
   - Waypoint list with up/down reorder buttons (custom UI on LRM
     `setWaypoints` — LRM has no native reorder)
5. **Incremental segment routing:** the waypoint list is split into consecutive
   pairs; each pair is one OSRM request (`overview=full&geometries=geojson`,
   always exactly 2 coordinate pairs per URL) with results cached per pair
   (6-decimal key). An edit fetches only the changed pairs, processed in
   ascending index order so snaps propagate. After each pair routes, both
   endpoint waypoints are replaced by the geometry's first/last vertex
   (**snap-and-freeze**) → adjacent segments share vertices exactly; waypoints
   always end up on the road centerline (user's click position discarded —
   the user sees the snapped result and can drag to adjust). Any waypoint,
   including the first, is freely editable. No leg UI exists.
6. Save/load: full-state JSON v2 (`waypoints` + `segments`); loading draws the
   route with zero network requests (segment cache seeded from the file).
7. Export → browser **save-as** dialog (Blob + `<a download>`).

### 3.3 Export contract (v1)

- **Format:** GeoJSON `Feature` with a single **merged `LineString`**
- **Content:** every vertex of the routed path (`overview=full`, 6-decimal
  WGS84), all segments concatenated, join vertices deduplicated
- **Properties:** `{exported, segments}` (export time / segment count —
  no semantic contract)
- **Not included:** way IDs, road attributes, per-segment structure, the user's
  clicked waypoints

### 3.4 Tech stack

| Area | Decision |
|---|---|
| App | Single `index.html` (HTML + CSS + JS inline), opened from `file://` |
| Map | Leaflet (vendored local `leaflet.js`/`leaflet-routing-machine.css` etc. — **no CDN**) |
| Routing UI | Leaflet Routing Machine **v3.2.12** (3.3.0 does not exist on npm), vendored; author calls it "barely maintained" → pinned local copy is the mitigation |
| LRM config | itinerary/address UI **off**, single route (no `alternatives`), `routeWhileDragging` off, waypoint markers + line-drag insertion on; `router` replaced by a custom segment router (OSRMv1 engine unused) |
| Routing backend | Public `https://router.project-osrm.org/route/v1/driving` (HTTPS, no key), one request per waypoint pair, max 2 concurrent, ~30 s timeout |
| Tiles | `https://tile.openstreetmap.de/{z}/{x}/{y}.png` (official `tile.openstreetmap.org` blocks `file://` requests — no Referer) |
| Assets/files | Folder: `index.html` + vendored Leaflet/LRM files + `track.csv` (test data). No server, no build step, no Docker |
| Large traces | `preferCanvas: true` + polyline simplification for the 50k-point ghost (display only; CSV untouched) |

### 3.5 Error handling

One actionable error banner. Failed pairs stay unrouted (previous segments keep
working; export stays possible for what is routed); a Retry action re-issues
routing for the current waypoint list. Messages per failure class:

- OSRM `400 NoRoute` / `NoSegment` / `InvalidQuery` → "no route between these
  waypoints — check waypoint placement/snapping"
- `429` / `5xx` / network failure / offline / IT block / timeout → "server
  unreachable — retry"
- Save/load validation (wrong version or shape, incl. old v1 files) →
  unsupported-file message, no Retry, state unchanged

(URL-too-long / waypoint-count warnings are gone: requests are always 2-pair.)

### 3.6 Explicit non-goals (v1)

- No waypoint auto-reordering, no route alternatives, no geocoding/address
  search, no turn-by-turn display, no statistics panels, no offline map data,
  no multi-user/server component.

---

## 4. Risks accepted

1. **Visual-only verification** — a wrong-branch route can look plausible at low
   zoom, yet this line becomes labeling ground truth. Cheap future insurance: a
   deviation-stats panel (not built).
2. **Way IDs deferred** — Python must still join exported geometry to OSM ways
   (trivial nearest-way match on clean geometry, but technically "matching",
   contrary to the original 'no re-matching' wish). v2 way-ID export removes it.
3. **External dependencies** — tool is dead if `router.project-osrm.org`,
   OSM tiles, or `file://` JS loading are blocked by enterprise IT. No
   fallbacks chosen.
4. **LRM unmaintained** — vendored, pinned, verified features (`linetouched`
   via-insertion) work against current Leaflet; risk contained to the pinned
   copy.
5. **Public OSRM etiquette** — no SLA, low request rates; manual human clicking
   keeps volume low by design; segment cache + 2-waypoint requests keep each
   request tiny and re-visits only changed pairs.

---

## 5. Acceptance test

1. Generate a dummy test CSV: a handful of ordered lat/lon points in the
   Thurgau–Zurich area (route output is meaningless, but exercises the whole
   workflow — the imported file is a visual aid only).
2. Open `index.html` from `file://`; ghost line renders, map fits bounds.
3. Place start + destination → route appears.
4. Add a via, drag the route line to insert another via, reorder waypoints;
   verify each edit fetches only the affected pairs and adjacent segments share
   exact vertices afterwards.
5. Drag the first waypoint (fully editable); save state to JSON, reload the
   page, load it back — route restored with zero network requests.
6. Export → save-as GeoJSON.
7. Re-open the exported file in Python (e.g. `json`/`geopandas`) and confirm a
   single merged LineString with deduped joins; plot it against the ghost CSV.

Pass = all steps work without console errors.
