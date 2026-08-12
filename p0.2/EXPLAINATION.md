# EXPLANATION.md — every file, every function, in plain language

This document explains the entire `blender_core_processing_p0.2/` codebase. It
assumes you can read code but does **not** assume you know Blender, 3D graphics,
computational geometry, or three.js. Every piece of jargon is explained the first
time it appears.

`README.md` is the "how do I use it and what did it measure" document. **This** is
the "how does every piece work, and why is it built this way" document.

---

## Table of contents

1. [What this program does](#1-what-this-program-does)
2. [Jargon dictionary](#2-jargon-dictionary)
3. [The big picture](#3-the-big-picture)
4. [The two orderings that carry the design](#4-the-two-orderings)
5. [`src/p02/coords.py` — the axis conversion](#5-coordspy)
6. [`src/p02/floors.py` — finding the storeys](#6-floorspy)
7. [`src/p02/rooms.py` — finding the rooms](#7-roomspy)
8. [`src/p02/signature.py` — recognising repeated plans](#8-signaturepy)
9. [`src/p02/classify.py` — object classes and the ratio solver](#9-classifypy)
10. [`src/p02/hotspots.py` — the viewer contract](#10-hotspotspy)
11. [`src/p02/report.py` — diagnostics](#11-reportpy)
12. [`src/p02/blender_locate.py` and `cli.py`](#12-blender_locate-and-cli)
13. [`blender/process_scene.py` — the orchestrator](#13-process_scenepy)
14. [`blender/bl_intake.py` — import and repair](#14-bl_intakepy)
15. [`blender/bl_geometry.py` — measurement](#15-bl_geometrypy)
16. [`blender/bl_render.py` — cubemaps and the self-test](#16-bl_renderpy)
17. [`blender/bl_optimize.py` — decimation, UVs, export](#17-bl_optimizepy)
18. [`website/` — the three.js viewer](#18-website)
19. [`tests/` — 146 tests](#19-tests)
20. [The data contracts, annotated](#20-the-data-contracts)
21. [A full run, traced](#21-a-full-run-traced)
22. [What it deliberately does not do](#22-what-it-deliberately-does-not-do)

---

## 1. What this program does

You give it a 3D building. **Nothing tells it where the floors or rooms are.** It
works that out from the shape of the geometry alone, then:

1. Figures out how many storeys the building has, and at what heights.
2. Notices which storeys have *identical* floor plans (a tower repeats one plan).
3. On each distinct plan, finds enclosed spaces and picks a good place to stand
   in each one.
4. Renders a 360° photo (a "cubemap") from eye height at each of those places.
5. Shrinks the model's triangle count without wrecking the delicate parts.
6. Exports a web-ready model and a JSON file describing every viewpoint.
7. Serves a browser viewer where each viewpoint is a clickable marker on the
   building, and clicking one drops you inside its 360° photo.

**P0.1 answered "how fast is it". P0.2 answers "is it a walkthrough".**

The payoff from step 2 is the largest single win in the codebase: on the test
building, 12 storeys share one plan, so **120 viewpoints cost only 10 renders**
instead of 120. Rendering is 97% of the runtime, so that is a 12× saving on
almost the entire cost of the program.

---

## 2. Jargon dictionary

Read this once and everything after it becomes easy.

### 3D basics

| Term | Meaning |
|---|---|
| **Mesh** | The shape of a 3D object: points in space plus instructions for which points join into flat surfaces. |
| **Vertex** (pl. **vertices**) | A single corner point. |
| **Polygon / face** | A flat surface joining 3+ vertices. |
| **Triangle** | A 3-corner face. Graphics cards can *only* draw triangles, so everything else is chopped up first. This is why we count triangles. |
| **N-gon** | A face with 5+ corners. One face, but several triangles to the GPU. |
| **Normal** | An arrow perpendicular to a surface, saying which way it faces. A floor's normal points up; a wall's points sideways. **This is how the program tells floors from walls.** |
| **Island / loose part** | A chunk of mesh not connected to the rest. A railing with 200 separate balusters is one object with 200 islands. |
| **Bounding box** | The smallest axis-aligned box containing an object. |
| **World space** | Coordinates in the scene's global frame, after an object's position/rotation/scale are applied. As opposed to **local space**, relative to the object's own origin. *Getting this wrong is a recurring theme below.* |
| **Decimation** | Deliberately deleting triangles to make a model smaller, while trying to keep it looking the same. |
| **UV map** | Instructions for wrapping a flat image onto a 3D surface, like a sewing pattern. Named UV because X, Y, Z were already taken. **No UVs means textures cannot be applied at all.** |

### File formats

| Term | Meaning |
|---|---|
| **FBX** | Autodesk's interchange format. Good at shapes and hierarchy, bad at modern materials. What 3ds Max exports. |
| **glTF / GLB** | The modern open format built for the web — "the JPEG of 3D". GLB is the single-file binary version. |
| **Draco** | Compression for 3D *geometry*. Does **not** compress textures. |
| **Cubemap** | A 360° panorama stored as six square images — the six faces of a cube around the viewer (+X, −X, +Y, −Y, +Z, −Z). Browsers can map these onto the inside of a cube so you can look around. |

### Algorithms used here

| Term | Meaning |
|---|---|
| **Histogram** | Counting how much of something falls into each of a series of ranges ("bins"). Used to find floor heights: bin all upward-facing surface area by height, and floors show up as spikes. |
| **Raster / rasterize** | Converting shapes into a grid of cells, like drawing on graph paper. |
| **Flood fill** | The paint-bucket tool. Start at a cell and spread to every connected neighbour that isn't a wall. |
| **Connected component** | A group of cells all reachable from one another. Each enclosed one is a candidate room. |
| **4-connected / 8-connected** | Whether "neighbour" means only up/down/left/right (4) or also diagonals (8). The difference matters enormously here — see [`flood_exterior`](#flood_exterior). |
| **Dilation** | Thickening shapes in a grid by one cell in all directions. Used to close hairline gaps in walls. |
| **Distance transform** | For every cell, how far is the nearest wall? |
| **Pole of inaccessibility** | The point inside a shape furthest from any edge. Geographically, it's the point in Antarctica furthest from the ocean. Here it's the best place to put a camera. |
| **Centroid** | The average position — the "balance point". **Not** the same as the pole, and for an L-shaped room the centroid falls *outside* the room. |
| **IoU** (Intersection over Union) | A similarity score for two shapes: overlap ÷ combined area. 1.0 = identical, 0 = no overlap. Used to decide two floors share a plan. |
| **Hash / digest** | A short fingerprint of some data. Identical data gives identical fingerprints, so comparing fingerprints is a fast way to compare data. |
| **Binary search** | Repeatedly halving a range to home in on a value. Used by the ratio solver. |
| **Newell's method** | A way to compute a polygon's area and normal that works for any flat polygon with any number of corners. |
| **Shoelace formula** | Area of a polygon from its corner coordinates. Named for the criss-cross pattern of the multiplication. |
| **Chamfer distance** | A fast approximation of straight-line distance on a grid, using integer steps (3 for straight, 4 for diagonal ≈ 3√2). |
| **Raycast** | Firing an imaginary line into the scene and asking what it hits first. |

### Blender

| Term | Meaning |
|---|---|
| **bpy** | Blender's Python library. `import bpy` only works *inside* Blender. |
| **bmesh** | Blender's lower-level mesh library, for per-vertex/per-face surgery. |
| **Headless / background** | Running Blender with no windows, driven by a script. |
| **Modifier** | A stacked, non-destructive effect — like a Photoshop filter layer. "Applying" it bakes it in permanently. |
| **Decimate modifier** | Blender's triangle reducer. Two modes matter here: **COLLAPSE** (merges vertices; good for flat things) and **DISSOLVE** (merges *coplanar* faces without moving anything; good for delicate things). |
| **EEVEE / Cycles** | Blender's two renderers. EEVEE is fast and approximate; Cycles is slow and physically accurate. |
| **Colour space** | Whether an image's numbers mean "brightness as the eye sees it" (**sRGB**) or "raw data" (**Non-Color**). Getting it wrong on a roughness map makes surfaces the wrong shininess everywhere. |

### Web / three.js

| Term | Meaning |
|---|---|
| **three.js** | The standard JavaScript 3D library. |
| **Sprite** | A flat image that always faces the camera. Used for the viewpoint markers. |
| **Import map** | A `<script type="importmap">` block telling the browser where bare names like `'three'` resolve to. Lets the code `import * as THREE from 'three'` with no build step. |
| **PMREM** | "Prefiltered Mipmapped Radiance Environment Map" — three.js's preprocessing that turns a background image into usable reflections. Without it, PBR materials look like flat grey plastic. |
| **Tone mapping** | Squashing bright values into displayable range. |
| **CORS** | Browser security rules that block `fetch` in certain situations. Why the viewer must be served over `http://`. |

---

## 3. The big picture

### Three worlds that cannot see each other

Same constraint as P0.1, and it dictates the whole layout:

> **Blender ships its own Python and cannot see uv-installed packages.**

```
   WORLD 1                  WORLD 2                    WORLD 3
   uv's Python              Blender's Python           Browser
   ───────────              ────────────────           ───────
   cli.py                   process_scene.py           app.js
   blender_locate.py        bl_intake.py               index.html
   tests/ (146)             bl_geometry.py             style.css
                            bl_render.py
                            bl_optimize.py
        │                          │                        │
        └──── subprocess ──────────┘                        │
                     │                                      │
                     └──→  output/*.glb                     │
                           output/hotspots.json  ───────────┘
                           output/report.json
                           output/cubemaps/**
```

The bridge is **`src/p02/`** — nine modules of pure standard-library Python,
imported by *both* World 1 and World 2. That is what makes 146 tests possible
without ever launching Blender.

The split is deliberate and consistent:

- **`src/p02/*`** — all the *thinking*. Floor detection, room finding, plan
  matching, classification, the solver. No `bpy` anywhere.
- **`blender/bl_*`** — all the *touching of Blender*. Measure the scene into
  plain tuples, hand them to `src/p02/`, act on the answers.

`bl_geometry.py` is the seam: it is the only module on the analysis path that
reads `bpy`, and everything it emits is plain data.

Two small files hold this together. `src/p02/__init__.py` is eight lines and
carries no code — just the package version and a docstring restating the
stdlib-only rule, so anyone opening the package first reads the constraint before
reading anything else. `pyproject.toml` declares `dependencies = []` (nothing at
runtime, `pytest` only in the dev group) and the `p02 = "p02.cli:main"` entry point
that makes `uv run p02` work.

### The stage pipeline

```
import → repair (weld) → measure → detect floors → slice each floor
      → group identical plans → find capture points → render cubemaps
      → classify → decimate → export GLB → write hotspots.json
```

---

## 4. The two orderings

Both are load-bearing. Reorder either and the program still runs, produces
plausible output, and is quietly much worse.

### Cubemaps render BEFORE decimation

The cubemaps are the photographic record of the building. Shooting them through a
model that has already lost 60% of its triangles throws away the thing you came
for. So `bl_render` runs at stage 6 and `bl_optimize` at stage 7.

### Plans are grouped BEFORE rendering

A residential tower repeats one plan a dozen times. Detecting that *first* turns
120 renders into 10. On the test model:

| | |
|---|---|
| Hotspots the viewer shows | **120** |
| Cubemaps actually rendered | **10** |
| Reused | **110** |
| Render time | 83 s of the 86 s total run |

Without the dedup, that 83 s becomes ~17 minutes.

---

## 5. `coords.py`

37 lines, and the most dangerous file in the codebase for its size.

**The problem:** Blender says **Z is up**. glTF (and therefore three.js) says
**Y is up**. Blender's exporter rotates the model on the way out. But hotspot
positions are computed in Blender coordinates and consumed by three.js — so if
the two disagree, *every marker floats somewhere plausible-looking and wrong,
with no error anywhere*.

```python
def blender_to_gltf(v):
    x, y, z = v
    return (x, z, -y)

def gltf_to_blender(v):
    x, y, z = v
    return (x, -z, y)
```

That is a −90° rotation about the X axis, matching exactly what Blender's
exporter does with its default `+Y Up` setting.

**The rule the docstring states, and the codebase obeys:** this is *the single
conversion point*. Nothing anywhere else may do this inline. `hotspots.py` calls
it, `bl_render.py` calls it, the self-test calls it. `app.js` never converts at
all, because positions arrive already converted.

---

## 6. `floors.py`

**Job:** given every upward-facing surface in the building, work out how many
storeys there are and at what heights. 310 lines, no `bpy`.

### The core idea

Floor slabs are large, flat, upward-facing, and repeat at regular heights. So:
**bin all upward-facing surface area by height, and storeys appear as spikes.**

```
 z=39.8 ▏█                        ← roof (a real slab, but nothing above it)
 z=36.3 ▏████████████
 z=33.3 ▏████████████             ← twelve clean spikes,
 z=30.3 ▏████████████                exactly 3.00 m apart
   ...
 z= 3.3 ▏████████████
 z= 0.0 ▏██████████████████████   ← site plane (much wider than the building)
```

### Inputs

```python
class HorizontalFace(NamedTuple):
    z: float          # height
    area: float       # WORLD-space area
    min_x, min_y, max_x, max_y: float   # footprint

class WallSpan(NamedTuple):
    z_min, z_max, area: float           # a near-vertical face's vertical extent
```

The `HorizontalFace` docstring carries a warning worth repeating: **`area` must be
computed from world-transformed vertices.** Blender's `polygon.area` is
*local-space* and silently wrong for any object carrying a scale — which, in a
3ds Max export, is all of them.

### `_histogram(faces, bin_size)`

```python
key = int(round(f.z / bin_size))
bins[key] = bins.get(key, 0.0) + f.area
```

Buckets every face's area by height. `BIN_SIZE = 0.1` (10 cm): finer just spreads
one slab across several bins; coarser starts merging a mezzanine into the storey
below.

### <a name="runs"></a>`_runs(bins, threshold, gap_bins)`

Groups above-threshold bins into contiguous runs, tolerating gaps up to
`RUN_GAP = 0.3` m.

**Why runs rather than peak-finding?** A slab whose top surface is split across
two adjacent bins produces *two* local maxima but is *one* level. Run-grouping
gets that right with no peak-prominence tuning to get wrong.

### `detect_levels(...)`

The main entry point.

```python
z = sum(f.z * f.area for f in member) / area
```

The level's height is the **area-weighted centroid** of its faces, not the bin
centre — so a level sitting between two bins lands where the geometry actually is.
This is why the test model reports `z = 3.3036` rather than a rounded `3.3`.

Then three cleanup passes:

#### `_merge_close_levels(levels, min_floor_height)`

Drops the smaller of any two levels closer than `MIN_FLOOR_HEIGHT = 2.0` m. A
concrete slab top and the finished floor 50 mm above it are one storey, not two.

#### `_tag_site(levels, factor)`

```python
if lv.extent_area > median * factor:
    lv.kind = "site"
```

The ground/landscape mesh is a big flat upward-facing surface — it looks exactly
like a floor to the histogram. It's caught by **XY extent**: it is much wider than
the building footprint. On the test model this correctly excludes a level at
z = 0.02.

#### `_tag_roof(levels, walls, storey_height, fraction)`

The roof is a *real* slab. The only thing distinguishing it from a floor is that
**nothing is enclosed above it**. So for each level, measure how much
near-vertical (wall) area exists in the storey-height band above it:

```python
lv.wall_area_above = wall_area_between(walls, lv.z + 0.05, lv.z + storey_height)
```

and demote levels with too little:

```python
for lv in reversed(candidates):        # top down ONLY
    if lv.wall_area_above < median * fraction:
        lv.kind = "roof"
    else:
        break
```

Two subtleties:

- **`reversed` and the `break`.** Demotion only ever walks down from the top and
  stops at the first level that *does* have walls above it. A mid-building level
  with little wall above it is an **atrium**, not a roof.
- **The threshold's provenance.** `ROOF_WALL_FRACTION = 0.45` is documented
  against real measurements: the twelve storeys each enclose ~2,297 units of wall,
  the roof only 720 (31%). It sits at half a storey rather than hugging either
  edge, because the cost of being wrong either way is one floor of hotspots.

Without this, a 12-storey building reports 13 floors and the viewer offers a
walkthrough of the sky.

### `wall_area_between(spans, z_lo, z_hi)`

```python
overlap = min(span.z_max, z_hi) - max(span.z_min, z_lo)
total += span.area * (overlap / height if height > 1e-9 else 1.0)
```

Pro-rates each wall by how much of it falls in the band, so a 40 m-tall facade
panel doesn't swamp a one-storey query.

---

## 7. `rooms.py`

**Job:** given a floor, find the rooms and pick where to stand in each. 496 lines,
no `bpy`, all integer grid work.

### The core idea

Take a horizontal slice through the building at eye height. Draw every wall that
crosses that slice onto graph paper. Flood-fill from outside. **Anything the fill
can't reach is enclosed — that's a room.**

```
before fill:              after exterior fill (~ = outside):
~~~~~~~~~~~~~~            ~~~~~~~~~~~~~~
.############.            ~############~
.#..........#.            ~#..........#~   ← the dots are unreachable
.#....##....#.            ~#....##....#~     from outside = a room
.#....##....#.            ~#....##....#~
.############.            ~############~
~~~~~~~~~~~~~~            ~~~~~~~~~~~~~~
```

### `class Grid`

An occupancy raster. `cells[x * height + y]` is 1 where a wall crosses.

```python
__slots__ = ("width", "height", "cell", "origin_x", "origin_y", "cells")
```

> **`__slots__`** tells Python not to give each instance a dictionary of
> attributes. Faster and much smaller — this class is created once per storey over
> potentially tens of thousands of cells.

`cells` is a `bytearray` — one byte per cell, contiguous in memory. `CELL = 0.25`
means each cell is 25 cm square.

#### `Grid.from_rows(rows)`

```python
y = height - 1 - row_i
```

Builds a grid from ASCII art, with `rows[0]` as the **top** row so what you write
looks like what you mean. This is a test helper, and it's why `tests/test_rooms.py`
reads like a floor plan:

```python
SIMPLE_ROOM = [
    "..............",
    ".############.",
    ".#..........#.",
    ".############.",
    "..............",
]
```

### `rasterize(segments, bbox, cell, margin_cells)`

Draws wall segments into the grid.

```python
steps = max(1, int(length / (cell * 0.5)) + 1)
```

**Half-cell steps.** A full-cell step can skip a diagonal cell and leave a
one-pixel hole — and one hole is all the flood fill needs to escape and merge
every room into the exterior.

`margin_cells = 2` guarantees a ring of free cells around the footprint, so the
exterior fill always has somewhere to start.

### `dilate(grid, iterations)`

Thickens walls by one cell in all 8 directions.

**This is the single most important line in room detection.** Real building
models have *coincident facade faces* — two surfaces at exactly the same place —
which leave hairline gaps at the rasterized resolution. Without dilation, the
exterior flood fill leaks through one of those gaps and swallows the entire
interior, and the program reports zero rooms on a building full of them.

### <a name="flood_exterior"></a>`flood_exterior(grid)`

Standard flood fill from every border cell, using an explicit stack rather than
recursion (a 100×100 grid would blow Python's recursion limit).

```python
for dx, dy in ((1, 0), (-1, 0), (0, 1), (0, -1)):   # 4-connected
```

**4-connected on purpose.** 8-connectivity would let the fill squeeze *diagonally*
between two wall cells that merely touch at a corner — which is exactly the leak
dilation exists to prevent. The two choices work as a pair: dilate 8-connected to
close gaps, fill 4-connected so nothing sneaks through corners.

### `enclosed_components(grid)`

Every free cell *not* reached by the exterior fill, grouped into connected blobs.
Each blob is a candidate room.

### `distance_transform(grid, component)`

For each cell in a blob, how far to the nearest non-blob cell (i.e. a wall).

Uses a **Chamfer 3-4 two-pass** algorithm: one forward sweep, one backward sweep,
with cost 3 for straight steps and 4 for diagonals (4/3 ≈ 1.33, close to √2 ≈ 1.41).
Divide by 3 at the end to get distance in cells.

```python
return {c: v / 3.0 for c, v in dist.items()}
```

**Why not a simple breadth-first search?** BFS on a 4-connected grid gives
*Manhattan* distance (city-block), which biases the result towards axis-aligned
corridors. Chamfer 3-4 lands within ~2% of true straight-line distance.

### `pole_of_inaccessibility(grid, component)`

The cell furthest from any wall.

```python
best = max(component, key=lambda c: (dist[c], -((c[0]-cx)**2 + (c[1]-cy)**2)))
```

A tuple key means: maximise distance first; **break ties** on nearness to the
centroid. Without the tiebreak, a symmetric rectangular room would pick an
arbitrary corner of the equidistant plateau instead of its middle.

**Why not just use the centroid?** Because for an L-shaped room the centroid falls
in the notch — *inside a wall*. The camera would then render the inside of a
brick. The pole is always inside the shape, and is also the most open place to
stand, which is exactly what you want a camera to do. `tests/test_rooms.py`
contains a guard test asserting that a centroid *would* have failed on the L-shape.

### `detect_rooms(regions, ...)`

Filters blobs down to plausible rooms:

| Constraint | Value | Rejects |
|---|---|---|
| `MIN_ROOM_AREA` | 4 m² | service ducts, gaps between facade panels |
| `MAX_ROOM_AREA` | 60 m² | atria, lobbies, undivided shells |
| `MIN_ROOM_DIMENSION` | 1.2 m | slivers and corridors |
| `MIN_CLEARANCE` | 0.4 m | points wedged against a wall |

### `viewpoint_grid(grid, regions, ...)`

The fallback when a floor has no room-sized spaces. Spreads capture points through
each too-large volume by **greedy farthest-first**: sort candidates by clearance
descending, accept one, then only accept further ones at least `GRID_SPACING = 8 m`
away.

Each point is returned as a single-cell `Region` carrying its parent's area, so
downstream code handles rooms and grid samples identically.

### `capture_points(grid, ...)` — the important design decision

```python
points = rooms + samples
```

**Rooms PLUS a grid, not rooms OR a grid.** Finding one room does not mean the
floor is covered.

The docstring explains exactly why, from the real fixture: the test building's
only room-sized enclosed spaces are two lightwell shafts between the wings. Its
four genuine volumes are 150, 146, 78 and 76 m² of undivided shell. Returning
just the two shafts would leave a 26×61 m floor plate with two capture points,
**both in service voids**. Grid-sampling the large volumes as well is what
actually covers the floor.

Returns `(points, summary)` where summary is `"rooms"`, `"mixed"`, `"grid"` or
`"none"` — and each point carries its own `kind`, so the report and the viewer can
tell a detected room from a fallback sample even on a floor that has both.

---

## 8. `signature.py`

**Job:** notice that twelve storeys share one floor plan. 130 lines.

**This is where the render budget is won.**

### `plan_signature(floor_index, grid)`

```python
min_x = min(c[0] for c in occupied)
min_y = min(c[1] for c in occupied)
rebased = frozenset((x - min_x, y - min_y) for x, y in occupied)
```

Takes the occupied (wall) cells and **re-bases them on their own bounding box**,
so two identical plans at different world positions — or on grids of different
sizes — produce the same set. Then hashes the sorted cells with SHA-256.

### `iou(a, b)`

```python
return len(a.cells & b.cells) / len(a.cells | b.cells)
```

Intersection over union on the cell sets. 1.0 = identical.

**Why both a hash and IoU?** The hash is nearly free and catches perfect matches.
But real geometry has float noise and the occasional extra bollard, so identical-
looking floors rarely hash identically. On the test model, the twelve storeys
matched at **IoU 0.9937** — not 1.0, but far above the threshold.

`IOU_THRESHOLD = 0.98` is deliberately high: below it you would start merging a
2BHK with a 3BHK and reuse the wrong cubemaps.

### `group_floors(signatures, ...)`

Greedy clustering against group *representatives* rather than pairwise:

```python
for group, rep in zip(groups, reps):
    if sig.digest == rep.digest: ...          # try the free check first
    score = iou(sig, rep)
    if score >= iou_threshold: ...
```

Two benefits over pairwise comparison: it's O(floors × groups) instead of
O(floors²), and membership is **deterministic** — a floor always joins the
earliest group it matches, so the representative is always the lowest storey with
that plan. That matters because the representative is the floor that actually gets
rendered.

---

## 9. `classify.py`

**Job:** decide what kind of thing each object is, and how hard each may be
decimated. 273 lines.

### The name-regex reality check

`CLAUDE.md` §4 says name matching should catch 60–70% of objects. The module
docstring records what actually happened:

> On the P0.2 intake model it catches **zero** — every object is a 3ds Max default
> name (`Box002`, `Cone001`, `Line003`).

So the name pass is kept as a cheap pre-filter for scenes that *do* have names,
and **geometry heuristics carry the real weight.**

### `ObjectMetrics`

What the classifier needs, measured inside Blender **after welding**:

```python
name, triangles, vertices, dims, islands, median_island_min_dim, has_uv, materials
```

The docstring warns: `islands` and `median_island_min_dim` are *meaningless* on an
unwelded 3ds Max FBX, because every triangle is its own island.

Derived properties:

```python
@property
def flatness(self):        # min(dims)/max(dims). Below 0.05 it's a plane.
@property
def thin_axis(self):       # which axis is narrowest — X/Y = wall, Z = slab
```

### `classify(m, use_names=True)`

Order matters, and the docstring says why: *"Triviality first (nothing to gain),
then island structure (the expensive mistake to get wrong), then planarity, then
names, then a solid default."*

**1. Trivial** — under 200 triangles. A ground plane at 12 triangles has nothing
to give.

**2. Thin feature** — the class that matters most:

```python
if m.median_island_min_dim < THIN_ISLAND_DIMENSION and (
    m.islands >= THIN_MIN_ISLANDS or m.triangles >= THIN_MIN_TRIANGLES
):
```

Railings, mullions, balusters, trim. Collapse-decimating them is what visibly
wrecks an arch-viz model, and they are often the *bulk* of the triangles — on the
test model a single railing object is 115,872 of 191,310.

They are detected by **physical thickness**, not triangle density. `THIN_ISLAND_DIMENSION
= 0.20` m: a baluster is 37 mm, a window frame 50 mm, a facade mullion 20 mm; a
facade panel or floor slab is hundreds of millimetres and must not qualify.

The comment records a failed approach worth preserving:

> Triangle density was tried first and is worthless here. Every object in a 3ds Max
> FBX export arrives as disconnected triangles, so tris-per-island is 1 for the
> entire scene until the weld repair runs — and after welding it says nothing about
> thinness.

**3. Planar** — `flatness < 0.05`. Then `slab` if the thin axis is Z (a floor),
`wall` otherwise.

**4. Name hints** — the regex table, as a fallback.

**5. `solid`** — the default.

### `Policy` and the ratio solver

```python
class Policy(NamedTuple):
    preferred: float   # the ratio we'd use if budget weren't binding
    floor: float       # past this the class stops looking like itself
    tolerance: float   # how fast the solver may push it toward the floor
    strategy: str      # "collapse" | "planar" | "keep"
```

| Class | preferred | floor | tolerance | strategy |
|---|---|---|---|---|
| `slab` | 0.10 | 0.05 | 3.0 | collapse |
| `wall` | 0.12 | 0.06 | 3.0 | collapse |
| `thin_feature` | 0.75 | 0.45 | **0.4** | **planar** |
| `vegetation` | 0.15 | 0.10 | 2.5 | collapse |
| `furniture` | 0.45 | 0.25 | 1.5 | collapse |
| `decorative` | 0.55 | 0.30 | 1.5 | collapse |
| `solid` | 0.30 | 0.15 | 1.5 | collapse |
| `trivial` | 1.0 | 1.0 | **0.0** | keep |

#### `_ratio_at(policy, s)` — the clever bit

```python
return max(policy.floor, policy.preferred * (s ** policy.tolerance))
```

`s` is **solver pressure**, running from 1.0 (no pressure, everyone at preferred)
down to 0.0 (everyone pinned at their floor). Using `tolerance` as an *exponent*
makes tolerant classes give way first:

| At `s = 0.5` | tolerance | `s ** tolerance` | effect |
|---|---|---|---|
| slab | 3.0 | 0.125 | gives up 87% of its allowance |
| thin_feature | 0.4 | 0.758 | gives up only 24% |

**So slabs are spent before railings are touched** — automatically, from one
scalar. `trivial` has tolerance 0.0, so `s**0 == 1` pins it at 1.0 whatever the
solver does.

#### `solve_ratios(metrics, target_keep, ...)`

```python
def kept_at(s):
    return sum(m.triangles * _ratio_at(POLICIES[kind], s) for m, kind, _ in classified)
```

`kept_at` increases with `s`. Three cases:

1. Preferred ratios already meet the target → `s = 1.0`, no pressure needed.
2. Even all floors overshoot the target → `s = 0.0`, and **`limited = True`**.
3. Otherwise → 60 rounds of binary search for the `s` that hits the target.

Case 2 is what happens on the real model, and the docstring is emphatic about why
that is correct behaviour rather than a failure:

> A facade that is 83% thin louvre and mullion geometry cannot reach 30% without
> destroying the louvres — the remaining reduction would have to come from removing
> elements or swapping in LODs, not from simplifying them, and that is out of
> scope. When that happens the solver stops at the floors and `floor_limited` says
> so, **rather than quietly wrecking the model to hit a number.**

The measured result: requested 30% kept, landed at 41%, and emitted
`TRIANGLE_BUDGET_MISSED` explaining exactly that. 78K triangles is comfortably
inside `CLAUDE.md` §4's 80–120K budget, which is the number that actually matters.

---

## 10. `hotspots.py`

**Job:** build `hotspots.json`, the contract between pipeline and viewer. 183 lines.

Two responsibilities kept deliberately together **because they must agree**:

- **render jobs** — where a cubemap actually gets rendered. One per capture point
  per **plan group**.
- **hotspots** — where the viewer draws a marker. One per capture point per
  **floor**.

That asymmetry *is* the dedup: 10 jobs, 120 hotspots.

### `FACES`

```python
FACES = ("px", "nx", "py", "ny", "pz", "nz")
```

**Must** match three.js's `CubeTextureLoader`, which expects `[+X, −X, +Y, −Y, +Z, −Z]`.
`bl_render.py` builds its camera rotations from this same tuple, so there is one
ordering, defined once.

### `render_jobs(floors, groups, points_by_group, eye_height)`

One job per capture point per group, positioned on the group's **representative**
floor:

```python
position=(point.x, point.y, rep.z + eye_height)
```

Still in Blender space (Z-up) — this feeds the renderer, which lives in Blender.

### `build_document(...)`

Assembles the JSON. The key loop:

```python
for floor_index in sorted(group.floors):
    position = coords.blender_to_gltf((point.x, point.y, level.z + eye_height))
    ...
    "shares_cubemap_with": None if hotspot_id == source_id else source_id
```

Every floor gets its own hotspot with its own converted position, but floors
sharing a plan all point at the same `cubemap.dir`. The `shares_cubemap_with`
field records the provenance for debugging without the viewer needing to care.

The docstring states the design goal plainly: *"so the viewer stays dumb: it draws
what it is given and loads the directory it is told to."*

**Note the conversion happens here**, at the boundary, through `coords`. This is
the only place hotspot positions change axis convention.

---

## 11. `report.py`

Structurally identical to P0.1's — `Diagnostic`, `GeometryStats`, `Report`, the
`ok`/`partial`/`failed` rollup, always-serialisable. See P0.1's EXPLANATION.md for
the mechanics.

**What P0.2 adds:**

```python
self.floors, self.plan_groups, self.hotspots = [], [], []
self.objects = []            # per-object decimation records
self.classification = {}     # the solver's summary
self.cubemaps = {"rendered": 0, "reused": 0, "engine": None, ...}
```

And a vocabulary of diagnostic codes as module constants:

```python
NO_FLOOR_LEVELS_DETECTED = "NO_FLOOR_LEVELS_DETECTED"
NO_ROOMS_DETECTED_USING_GRID = "NO_ROOMS_DETECTED_USING_GRID"
HOTSPOT_OBSTRUCTED = "HOTSPOT_OBSTRUCTED"
TRIANGLE_BUDGET_MISSED = "TRIANGLE_BUDGET_MISSED"
UV_DISTORTED = "UV_DISTORTED"
TEXTURE_COLORSPACE_CORRECTED = "TEXTURE_COLORSPACE_CORRECTED"
...
```

> **Why constants instead of string literals?** A typo becomes an `ImportError`
> instead of a silently unmatched string in a test. The comment says exactly that.

One subtle contract, noted in `summary_lines()`:

```python
"""cli.py greps stdout for these prefixes, so the leading keywords are part
of the contract between the two interpreters."""
```

The words `status`, `triangles`, `floors`, `hotspots` at the start of each line are
matched by `cli.SUMMARY_KEYS`. Change one and the terminal output goes silent.

---

## 12. `blender_locate` and `cli`

### `blender_locate.py`

Identical to P0.1's: check `--blender`, then `$BLENDER_BIN`, then `PATH`, then
known install locations; verify each is both a file *and* executable
(`os.access(path, os.X_OK)`); raise `BlenderNotFound` with instructions if none
work. `blender_version()` returns `""` rather than raising on failure.

### `cli.py`

Also structurally the same as P0.1's, with more knobs:

| Group | Flags |
|---|---|
| geometry | `--target-ratio`, `--no-draco`, `--no-uv-transfer` |
| capture | `--eye-height`, `--cell`, `--engine`, `--samples`, `--cubemap-size`, `--cubemap-lo`, `--exposure`, `--skip-cubemaps`, `--no-auto-light` |
| viewer | `--serve`, `--port`, `--no-open`, `--serve-only` |
| test | `--selftest-cubemap` |

Points worth noting:

```python
p.add_argument("input", nargs="?", ...)
```

`nargs="?"` makes the input optional, because `--selftest-cubemap` and
`--serve-only` need no model.

```python
--exposure default=-0.4
```

Documented as: *"Interiors lit only by sky bleed blow out at 0; the default pulls
them back."* A measured constant, not a guess.

```python
handler = functools.partial(http.server.SimpleHTTPRequestHandler,
                            directory=str(PACKAGE_ROOT))
```

Serves the **package root**, not `website/`, because the page loads from
`../output/`. A web server can't serve above its root, so both directories must
sit under one document root.

```python
class ReusableTCPServer(socketserver.TCPServer):
    allow_reuse_address = True
```

Without this, the port stays reserved by the OS for a minute or two after
stopping (TCP `TIME_WAIT`) and an immediate restart fails.

Exit codes: `0` success, `1` processing failed, `2` bad arguments.

---

## 13. `process_scene.py`

**The orchestrator.** Runs inside Blender, 312 lines, and reads like the pipeline
diagram.

### The import dance

```python
sys.path.insert(0, str(repo_root / "src"))
sys.path.insert(0, str(repo_root / "blender"))

import bpy
import bl_render
from p02 import report as rp
```

The module docstring explains: *"Imports of `p02` and the `bl_*` modules happen
inside functions, after `--repo-root` has been added to `sys.path`. Blender runs
this file directly, so there is no package context to import from at module load."*

That's why the imports look misplaced — they can't be at the top.

### `main()` — the safety net

```python
try:
    _run(args, report, outdir, input_path)
except Exception as exc:
    report.fatal_error = f"{type(exc).__name__}: {exc}"
    traceback.print_exc()

report.write(report_path)      # OUTSIDE the try
```

Whatever explodes, a valid `report.json` is written. That file is the only artifact
the caller gets.

### `_run(...)` — the eight stages

Each is timed with `report.time_phase(...)`.

**1. Intake** — `bl_intake.prepare()`. Note the search roots:

```python
search_roots = [input_path.parent, input_path.parent.parent]
```

Textures usually live beside or one level above the model.

**2. Measure** — `bl_geometry.collect()` walks every mesh **once** and caches
everything downstream needs.

**3. Floors** — `detect_levels(horizontal_faces, wall_spans)`.

**4. Slice and group:**

```python
for level in storeys:
    segments = bl_geometry.wall_segments_at(geometries, level.z + args.eye_height)
    grid = dilate(rasterize(segments, footprint, cell=args.cell), 1)
    grids[level.index] = grid
    signatures.append(plan_signature(level.index, grid))

groups = group_floors(signatures)
```

Note it slices at `level.z + eye_height` — 1.6 m above the slab, where a person's
head is, not at floor level where door thresholds would confuse things.

**5. Capture points** — only on each group's *representative* floor, with a
warning tailored to what was found (`grid`, `mixed`, or `none`).

**6. Cubemaps** — `bl_render.ensure_lighting()` then `render_cubemaps(jobs, ...)`.

**7. Classify and decimate** — `solve_ratios()` then `bl_optimize.optimize()`.

**8. Export** — two GLBs plus `hotspots.json`.

### `_verify_clearance(jobs, report, rp)`

A quality gate worth understanding:

```python
directions = [(1,0,0), (-1,0,0), (0,1,0), (0,-1,0), (0,0,1), (0,0,-1)]
for job in jobs:
    for direction in directions:
        hit, location, *_ = bpy.context.scene.ray_cast(depsgraph, origin, direction)
        ...
    if nearest < 0.4:
        report.warning(rp.HOTSPOT_OBSTRUCTED, ...)
```

**Why this exists:** the room grid scores clearance in **2D**. A low soffit or a
mezzanine slab directly overhead is completely invisible to a horizontal slice. A
six-direction ray fan catches those before you waste a render on the inside of a
ceiling.

---

## 14. `bl_intake.py`

**Job:** import the file and repair it. 386 lines. Nothing here aborts the run.

### `import_model(path, report)`

Dispatches on extension: `.fbx`, `.glb`/`.gltf`, `.blend`, `.obj`. OBJ emits a
warning because it *"carries no hierarchy and no material assignments"* — it's a
last-resort fallback per `CLAUDE.md` §7.

### <a name="weld"></a>`repair_geometry(report)` — the most important function in P0.2

```python
bmesh.ops.remove_doubles(bm, verts=bm.verts, dist=WELD_DISTANCE)   # 1e-4
bmesh.ops.dissolve_degenerate(bm, dist=WELD_DISTANCE, edges=bm.edges)
loose = [v for v in bm.verts if not v.link_edges]
```

**The discovery this codebase is built around**, from the module docstring:

> A 3ds Max FBX export arrives with its meshes exploded into disconnected
> triangles: on the P0.2 intake model, one object has **115,872 triangles across
> 115,872 separate islands and 347,616 vertices for what is really 58,752**.

Every triangle is its own island. Nothing is joined to anything.

Two consequences, both fatal:

1. **Decimation does nothing.** Collapse works by merging vertices across shared
   edges. With no shared edges, there is nothing to collapse.
2. **Every island-based heuristic reads garbage.** `tris_per_island` is 1 for the
   entire scene, so `classify()` cannot tell a railing from a wall.

Welding costs **0.9% of triangles** and fixes both. The README's verdict: *"It is
the single most important repair step, and it is not in `CLAUDE.md`'s pipeline
description."*

Each object is welded in its own try/except, so one pathological mesh can't take
the rest down.

### `resolve_textures(search_roots, report)`

Builds an index of every image file under the search roots, keyed by **stem
lowercased** (filename without extension):

```python
index.setdefault(candidate.stem.lower(), candidate)
```

Then for each image that genuinely failed to load, looks up its stem and relinks.

**Why match on stem rather than full filename?** Because *"agencies routinely
re-save .jpg as .jpeg"*.

Note the early-out:

```python
if image.source == "GENERATED" or image.packed_file is not None: continue
if image.has_data: continue
```

Textures embedded in the FBX arrive already packed and need nothing. On the test
model every recorded path points at a `New folder (42)` that doesn't exist — and
it doesn't matter, because the pixels came in embedded.

### <a name="colorspace"></a>`_colorspace_verdict(node)` and `fix_colorspaces(report)`

**The best cautionary tale in the codebase.**

The problem: Blender's FBX importer marks *every* texture as sRGB. A roughness or
metalness map read through an sRGB curve gives the wrong surface response
everywhere. So they need correcting to `Non-Color`.

The obvious fix — check the filename for `roughness`, `metalness`, `normal` — is
**worse than the disease**. The docstring records what happened:

> Measured on the P0.2 intake model: five of its eight textures are named
> `*_AmbientOcclusion`, `*_Displacement` and `*_METALNESS` — and every one of them
> is wired into **Base Color**. The artist dragged whatever map looked right into
> the diffuse slot... An earlier version of this function trusted the name and
> **corrupted the diffuse of most of the building**.

That is precisely the drag-and-drop library-material behaviour `CLAUDE.md` §1
describes.

The fix: decide from **how the texture is wired**, not what it is called.

```python
if target.type in DATA_CONSUMER_NODES:        # Normal Map, Bump, Displacement
    verdicts.append(True)                     # definitely data
elif target.type in SHADER_NODES:
    verdicts.append(link.to_socket.name not in COLOR_SOCKETS)
```

Three refinements worth noting:

- **The Alpha output is skipped.** It's a separate channel and says nothing about
  the RGB's colour space.
- **`DATA_CONSUMER_NODES` is checked first.** A Normal Map node's input socket is
  *also* called "Color", so socket-name logic alone would get it backwards.
- **Colour wins a disagreement:** `return all(verdicts)`. One texture feeding both
  Base Color and Roughness must stay sRGB, because *"a diffuse read as data is
  glaring, a roughness read as colour is subtle."*

The filename regex survives only as a last resort when nothing conclusive is
wired. Even then it's carefully written:

```python
r"(?<![a-z])(normals?|nrm|roughness|...|ao|occlusion|orm|...)(?![a-z])"
```

Those lookarounds are boundary anchors. The comment explains: *"unanchored, `ao`
matches half the words in English and `orm` matches `platform`, `normal` and
`transform`."*

The report records `decided_by: "wiring"` or `"name"` so you can audit which path
each decision took.

### `ensure_materials(report)`

Any mesh with no material gets a shared default grey and a warning. A degraded
result beats a tool that refuses to run.

---

## 15. `bl_geometry.py`

**Job:** measure the scene into plain data. 274 lines. The only module on the
analysis path that touches `bpy`.

### The world-space warning

The module docstring:

> World space throughout. Blender's `polygon.area` and `polygon.normal` are
> **LOCAL**, and silently wrong for any object carrying a scale — which, in a
> 3ds Max export, is all of them.

### `collect(objects)` — the single pass

```python
normal_matrix = matrix.to_3x3().inverted_safe().transposed()
world = [matrix @ v.co for v in mesh.vertices]
```

> **Why the inverse transpose for normals?** Under non-uniform scale, normals do
> *not* transform like positions. Squash a sphere flat and its surface normals
> must tilt the *opposite* way to stay perpendicular. The inverse transpose of the
> 3×3 matrix is the correct transform. Using the plain matrix would misclassify
> tilted faces on any scaled object.

Then each polygon is sorted by its normal's Z component:

```python
if normal.z > UP_NORMAL_MIN:            # 0.9  → floor-like
    geometry.horizontal.append(HorizontalFace(...))
elif abs(normal.z) <= WALL_NORMAL_MAX:  # 0.35 → wall-like
    geometry.walls.append(WallSpan(...))
    geometry.wall_polygons.append(points)
```

**The dead band between 0.35 and 0.9 is intentional** — soffits, ramps and
chamfers are counted as neither, rather than being forced into a category where
they'd pollute the histogram.

The `ObjectGeometry` docstring explains why everything is cached: *"the room slice
runs once per storey, and re-deriving world coordinates twelve times over 190,000
polygons is the difference between seconds and minutes."*

### `_polygon_area(points)` — Newell's method

```python
normal.x += (a.y - b.y) * (a.z + b.z)
normal.y += (a.z - b.z) * (a.x + b.x)
normal.z += (a.x - b.x) * (a.y + b.y)
return normal.length / 2.0
```

Computes a polygon's area from its world-space corners, for any number of corners,
in any orientation. This is the correct replacement for the local-space
`polygon.area`.

### `_island_stats(obj)`

Walks the mesh graph to find loose parts, then measures each one's narrowest
dimension:

```python
spans = [(max(p[i] for p in points) - min(p[i] for p in points)) * abs(scale[i])
         for i in range(3)]
thicknesses.append(min(spans))
```

Returns `(island_count, median_thickness)`. This is the *"37 mm baluster vs 400 mm
facade panel"* measurement that drives `thin_feature` classification. **Only
meaningful after welding.**

### `building_footprint(geometries, site_extent_factor)`

XY bounds with the site plane excluded, by dropping objects whose footprint area
exceeds `median × factor²`. Including a landscape mesh would make the room raster
mostly empty ground — slow, and it drags the exterior flood fill into places it
should never reach.

### <a name="slicing"></a>`wall_segments_at(geometries, z)` — true plane slicing

For every near-vertical polygon that spans height `z`, compute where it crosses:

```python
if (a.z - z) * (b.z - z) > 0:
    continue                            # both ends same side, no crossing
t = (z - a.z) / (b.z - a.z)             # how far along the edge
crossings.append((a.x + (b.x - a.x) * t, a.y + (b.y - a.y) * t))
```

> **The sign trick:** if `(a.z - z)` and `(b.z - z)` have the same sign, their
> product is positive and both endpoints are on the same side of the plane. Only
> edges whose product is ≤ 0 actually cross it.

**Why the true intersection rather than projecting the polygon down?** The comment
is explicit: projecting *"would paint a wall's whole height into the raster and
close off doorways that the slice passes cleanly through."* A doorway is a gap in
the wall at head height; project the wall's outline downward and the doorway
disappears, walling off rooms that are actually connected.

The performance note also matters:

```python
for span, points in zip(geometry.walls, geometry.wall_polygons):
    if span.z_min > z or span.z_max < z:
        continue
```

Rejecting on the cached Z span first *"turns a 190,000-polygon scan into a few
thousand, which matters because this runs once per storey."*

---

## 16. `bl_render.py`

**Job:** render a cubemap at each capture point. 383 lines.

### `FACE_ORIENTATION_GLTF` — the highest-risk constant in P0.2

```python
FACE_ORIENTATION_GLTF = {
    "px": (( 1, 0, 0), (0, 1, 0)),
    "nx": ((-1, 0, 0), (0, 1, 0)),
    "py": (( 0, 1, 0), (0, 0, 1)),
    "ny": (( 0,-1, 0), (0, 0,-1)),
    "pz": (( 0, 0, 1), (0, 1, 0)),
    "nz": (( 0, 0,-1), (0, 1, 0)),
}
```

Each entry is `(forward, up)` for one cube face, in **glTF axes**.

The docstring states the danger precisely: *"A wrong entry does not raise — it
produces six plausible images that tile into a broken panorama."* You get six
perfectly good JPEGs that simply don't join up.

The vertical faces follow the OpenGL cubemap convention; the four side faces keep
world-up at the top of the image so the JPEGs are also human-readable when opened
directly.

### `look_at(forward, up)`

Builds a rotation matrix aiming a camera:

```python
z = -f                    # Blender cameras look down local -Z
x = u.cross(z)
y = z.cross(x)
return Matrix((x, y, z)).transposed()
```

> **Why `-forward`?** Blender cameras look along their local **−Z** axis with
> local **+Y** up. So the basis is `(right, up, −forward)` as columns, and
> `.transposed()` converts the rows we built into columns.

There's a degenerate-case guard: if `forward` is parallel to `up`, the cross
product collapses to zero length, so it picks a different reference axis.

### `ensure_lighting(report, force=False)`

```python
if scene_has_lights() and not force:
    return False
```

Adds a Nishita physical sky and a sun when the scene has none.

> **Why this is needed:** *"Marketplace exterior models routinely ship with no
> lights at all — the P0.2 intake model has seventeen meshes and nothing else.
> Rendering it as-is gives six black images."*

```python
light_data.energy = 2.0
```

Deliberately not a daylight-accurate figure. The comment: *"these are interiors lit
only by sky bleeding through window openings, and a physical sun value washes the
pale concrete and tile in this model straight to white."*

### `configure_render(scene, engine, size, samples, exposure)`

Square output, JPEG at quality 85, and for EEVEE:

```python
for flag in ("use_raytracing", "use_gtao", "use_ssr"):
```

Ray tracing, ambient occlusion and screen-space reflections on — *"makes interior
bounce light usable rather than flat, which matters when the only light source is
the sky coming through window openings."* Each is set inside a `hasattr`/`try`
because the flags differ between Blender versions.

### `make_camera(size)`

```python
data.angle = math.radians(90.0)
data.sensor_fit = "AUTO"
```

**90° field of view is not a choice** — it's what makes six faces tile into a
seamless cube. `sensor_fit = "AUTO"` on a square render means that one angle
covers both axes.

### `render_cubemaps(jobs, outdir, report, ...)`

For each job, for each of the six faces: aim the camera, render, write
`{face}.jpg`, then produce a 256 px preview.

```python
camera.matrix_world = Matrix.Translation(position) @ look_at(forward, up).to_4x4()
```

Position and rotation combined into one transform.

### `_write_downscaled(source, target, size)`

Makes the low-res preview by **scaling the render, not re-rendering it**. The
256 px set exists so the jump into a viewpoint is instant; re-rendering would
double the most expensive stage for no benefit.

### The self-test — `build_selftest_scene()` and `selftest_cubemap()`

Since a wrong orientation entry can't raise, this is the only way to be sure.

It builds a room with six differently-coloured emissive walls, renders all six
faces, samples the centre pixel of each, and asserts the right colour appears.

Three design decisions, each guarding against a *different* way the test could
lie:

**1. A cube, not six placed planes.**
> *"a plane needs an orientation, and getting **that** wrong would produce failures
> indistinguishable from the bug this test exists to catch."*

A cube's faces come with correct normals for free.

**2. Wall colours assigned through the same `coords` conversion:**

```python
direction_to_face = {
    tuple(round(c) for c in coords.gltf_to_blender(FACE_ORIENTATION_GLTF[f][0])): f
    for f in FACES
}
```

If `coords` had a sign error, a test that hardcoded its own mapping would cancel
the error out and pass. Routing through the real conversion means a bug there
surfaces *here*.

**3. View transform pinned to Standard:**
```python
scene.view_settings.view_transform = "Standard"
```
> *"Blender's default AgX desaturates saturated emission far enough that cyan reads
> as white and the test cannot tell a correct face from a wrong one."*

And the comparison is channel-presence, not exact values:

```python
got = tuple(1.0 if c > 0.25 else 0.0 for c in sampled)
```

*"Compare which channels are lit rather than exact values — JPEG is off the table
here but gamma still is not."*

---

## 17. `bl_optimize.py`

**Job:** decimate, keep UVs alive, export. 317 lines.

### Two strategies

```python
def _collapse(obj, ratio):
    modifier.decimate_type = "COLLAPSE"
    modifier.ratio = max(0.0, min(1.0, ratio))
    modifier.use_collapse_triangulate = True

def _planar_dissolve(obj):
    modifier.decimate_type = "DISSOLVE"
    modifier.angle_limit = PLANAR_ANGLE          # 5°
    modifier.use_dissolve_boundaries = False
```

- **COLLAPSE** merges vertices — it *moves geometry*. Correct for slabs and walls,
  where losing 90% of a planar surface is invisible.
- **DISSOLVE** merges faces that are within 5° of coplanar — it **moves nothing**.
  It's *lossless* for flat regions: you're deleting edges that describe no shape.

`use_dissolve_boundaries = False` protects the outer edges of each surface, which
are exactly what defines a thin part's silhouette.

### The planar path — dissolve first, collapse only if needed

```python
ok = _planar_dissolve(obj)
after_dissolve = triangle_count(obj)
wanted = int(before * plan.ratio)
if ok and after_dissolve > wanted and after_dissolve > 0:
    ok = _collapse(obj, wanted / after_dissolve)
    record["applied"] = "planar+collapse"
```

Lossless work first; only pay for lossy work if lossless didn't get there.

**Measured on the real model** — this is the headline result of P0.2:

| object | class | applied | before | after | keep |
|---|---|---|---|---|---|
| Cone001 | thin_feature | **planar only** | 115,872 | 37,536 | **0.32** |
| Shape | thin_feature | planar+collapse | 32,994 | 21,830 | 0.66 |
| Line003 | solid | collapse | 17,260 | 11,660 | 0.68 |
| Box002 | thin_feature | planar+collapse | 7,704 | 3,466 | 0.45 |
| Line005 | solid | collapse | 7,008 | 1,050 | 0.15 |

`Cone001` — 37 mm balusters, 60% of the entire scene — dropped to 32% kept
**through lossless coplanar merging alone**, beating its 0.45 floor without
collapsing a single element.

### `uv_area(obj)` — the corruption detector

```python
accumulator += u0 * v1 - u1 * v0    # shoelace formula
total += abs(accumulator) / 2.0
```

Total area covered in UV space.

> **Why measure this?** *"Triangle counts stay perfectly plausible while the
> mapping underneath is destroyed, so this is what catches it."* A model can have
> exactly the right triangle count and completely scrambled textures.

Checked against sane bounds afterwards:

```python
UV_AREA_SANE_MIN = 0.35     # decimation legitimately shrinks UV area a little
UV_AREA_SANE_MAX = 1.15     # near 1.0 because nothing legitimate GROWS a layout
```

### <a name="uvstory"></a>The UV transfer story — a bug found by measurement

This is the most instructive comment block in P0.2. `CLAUDE.md` §4 prescribes a
Data Transfer modifier to project original UVs onto the decimated mesh. P0.2
**does not do that on the happy path**, and the code explains why:

```
# Decimate interpolates UVs itself, and measurement says it does the job:
# on Cone001 (115,872 tris) collapse changed total UV area by 0.0%.
#
# The Data Transfer pass this used to run unconditionally on top of that
# was destroying them. Measured, same decimation, with vs without:
#
#     Line003    -16.0%  ->  +21311.5%
#     Cone001     +0.0%  ->    +222.0%
#     Box414     -45.7%  ->    +173.0%
```

A **21,311%** UV area increase. The cause:

> `POLYINTERP_NEAREST` samples the nearest source polygon, which is incoherent once
> UVs tile — and this model tiles 27–55× (`Line003` spans u[−13.39, 14.39]).
> Adjacent destination loops land on different tiles and the mapping explodes.

> **UV tiling:** UV coordinates outside 0–1 make the texture repeat. A wall with
> u from −13 to +14 tiles its texture 27 times. Two neighbouring points on the
> decimated mesh can sample source faces on *different tiles* — one at u=2.1,
> one at u=3.9 — and interpolating between them smears the texture across the
> whole range.

So the transfer became **recovery only**:

```python
if had_uv:
    if len(obj.data.uv_layers) == 0:        # decimation actually dropped them
        record["uv_tier"] = "A-restored"
        if donor is None or not _transfer_uvs(obj, donor): ...
    else:
        record["uv_tier"] = "A"             # they survived; leave them alone
```

It runs *only* when decimation genuinely dropped the UV layer, never to "improve"
UVs that survived.

The three UV tiers recorded per object:

| Tier | Meaning |
|---|---|
| `A` | Had UVs, they survived decimation. The common case. |
| `A-restored` | Had UVs, decimation dropped them, transferred back from the donor. |
| `A-failed` | Had UVs, lost them, could not recover. An error. |
| `B` | Never had UVs; Smart UV Project at 66°. |
| `B-failed` | Never had UVs and unwrapping failed. |

### `export_glb(path, report, draco)`

```python
export_yup=True,   # glTF convention; p02.coords assumes it
```

That comment is the link back to [`coords.py`](#5-coordspy). If this flag ever
changed, `coords.blender_to_gltf` would silently become wrong and every marker
would be misplaced.

---

## 18. `website/`

### `index.html`

```html
<script type="importmap">
{ "imports": { "three": "./vendor/three.module.js", "three/addons/": "./vendor/" } }
</script>
```

> **Import map.** Normally `import * as THREE from 'three'` requires a bundler to
> resolve the bare name `'three'`. An import map tells the browser directly, so
> the viewer runs with **no build step and no node_modules** — which is what lets
> it work fully offline from vendored files.

The rest is chrome: a boot overlay, a fade layer for transitions, a floor
selector, a legend, a stats box, a tooltip, a pano-mode bar, and an error panel
that tells you to serve over HTTP.

### `app.js` — 566 lines

**Two modes over one scene:**

- **building** — orbit the GLB; every capture point is a marker.
- **pano** — stand at a capture point inside its cubemap.

#### `makeSkyTexture()`

Draws a two-stop vertical gradient on a 16×256 canvas and uses it as both the
background **and**, via `PMREMGenerator`, the scene environment.

> **Why an environment matters:** PBR materials calculate reflections from their
> surroundings. With no environment, metal and glossy surfaces have nothing to
> reflect and read as *flat grey plastic*. The pipeline ships no HDRI, so the
> viewer synthesises one.

#### `makeMarkerTexture(color)`

Draws a ring with a solid centre on a canvas, once per marker kind, shared by
every sprite. Two textures total rather than 120.

#### `buildMarkers(hotspots)`

```python
depthTest: false, depthWrite: false
sprite.material.sizeAttenuation = false
sprite.renderOrder = 999
```

- `depthTest: false` — markers always draw on top. Occlusion is communicated by
  *dimming* instead, *"so a viewpoint behind a wall is still discoverable."*
- `sizeAttenuation = false` — markers stay the same size on screen regardless of
  distance, like a UI element rather than an object.

#### `pickMarker(px, py)` — screen-space, not raycast

```javascript
const distance = Math.hypot(point.x - px, point.y - py);
if (distance < bestDistance) { ... }
```

The comment explains: *"sprite raycasting does not account for
`sizeAttenuation: false`, so the pickable area would drift away from the drawn
marker as the camera moves."* Since the markers are drawn at fixed screen size,
they must be *picked* at fixed screen size too.

#### `updateOcclusion()`

```javascript
if (now - state.lastOcclusion < OCCLUSION_INTERVAL_MS) return;   // 160ms
...
occlusionRay.far = distance - 0.25;
const blocked = occlusionRay.intersectObjects(state.buildingMeshes, false).length > 0;
marker.sprite.material.opacity += (target - marker.sprite.material.opacity) * 0.5;
```

Fires a ray from camera to marker; if the building blocks it, fade toward 28%
opacity. Throttled to ~6 Hz because it's a raycast per visible marker — *"smooth
enough to read as lighting and cheap enough to ignore."* The `far = distance -
0.25` stops the marker's own surroundings counting as an occluder.

#### `loadCubemap(hotspot, token)` — progressive loading

```javascript
loader.load(faces.map((f) => `${f}.lo.${ext}`), apply, undefined, () => {});
loader.load(faces.map((f) => `${f}.${ext}`), apply, undefined, () => { ... });
```

Both requests fire at once; the 256 px set arrives first and shows immediately,
the 1024 px set swaps in underneath.

```javascript
const apply = (texture) => {
  if (token !== state.panoToken) return;      // user already moved on
  ...
};
```

> **The token pattern.** Every mode change increments `state.panoToken`. A slow
> high-res load that lands after the user has left is discarded rather than
> stamping a stale panorama over the current view. This is the standard fix for
> out-of-order async results.

#### `enterPano(hotspot)` / `exitPano()`

```javascript
camera.position.copy(position);
controls.target.copy(position).add(heading);      // target 0.1m in front
controls.enableZoom = false;
controls.rotateSpeed = -0.35;                     // inverted
```

`OrbitControls` orbits *around a target*. To make it behave like looking around
from a fixed point, the target is placed just 10 cm in front of the camera — so
"orbiting" it barely moves the camera but does swing the view.

`rotateSpeed = -0.35` is negative deliberately: *"dragging should push the view"*,
which is the opposite sense from orbiting an object.

#### Click vs drag

```javascript
const moved = Math.hypot(event.clientX - pointerDownAt.x, event.clientY - pointerDownAt.y);
if (moved > 6) return;
```

Moving more than 6 px between press and release means you were orbiting, not
clicking. Without this, every drag that happens to end over a marker would teleport
you into a panorama.

#### `frameModel(object)`

```javascript
const distance = radius / Math.sin((camera.fov * Math.PI) / 360);
```

Trigonometry to find the camera distance at which an object of the given radius
exactly fills the field of view. `fov/360` in radians is `fov/2` in degrees — half
the field of view, which is the angle in the right triangle.

Also sets `near`/`far` from the model size, which prevents **Z-fighting** (flickering
surfaces caused by insufficient depth precision) on large buildings.

### `vendor/`

three.js r180, committed: `three.module.js`, `three.core.js`, `GLTFLoader`,
`DRACOLoader`, `OrbitControls`, `BufferGeometryUtils`, and the Draco decoder
(`.js` + `.wasm`).

> **`.wasm`** = WebAssembly, a compiled binary format that runs at near-native
> speed in browsers. Draco decoding is maths-heavy, so it ships compiled.

Vendored so the viewer works with no network at all.

---

## 19. `tests/`

**146 tests, no Blender required.** That is only possible because `src/p02/` is
dependency-free.

| File | Tests | Covers |
|---|---|---|
| `test_classify.py` | 36 | thin-feature detection, planarity, name hints, the solver |
| `test_rooms.py` | 32 | flood fill, L-shapes, hairline leaks, slivers, corridors, the pole |
| `test_hotspots.py` | 19 | render dedup, document structure, coordinate conversion |
| `test_floors.py` | 17 | peak finding, site plane, parapet roof, mezzanine merging |
| `test_report.py` | 17 | status rollup, serialisation |
| `test_signature.py` | 16 | plan hashing, IoU, grouping determinism |
| `test_coords.py` | 9 | the axis round-trip |

Two things make this suite unusually valuable:

**1. Grids are written as ASCII art.**

```python
SIMPLE_ROOM = [
    "..............",
    ".############.",
    ".#..........#.",
    ".############.",
]
```

The test reads the way the floor plan looks, so a failure is diagnosable by eye.

**2. The classifier is tested against the real measured metrics of all 17 intake
objects.** So a threshold change that would wreck the fixture fails in pytest in
0.05 s, rather than three minutes into a Blender run.

There is also a guard test asserting that a **centroid would have failed** on the
L-shaped room — it pins the *reason* the pole of inaccessibility exists, not just
its current behaviour.

Deliberately not covered by pytest: anything requiring Blender. Cube orientation
is covered by `--selftest-cubemap`; the viewer needs a browser.

---

## 20. The data contracts

### `hotspots.json` — pipeline → viewer

```jsonc
{
  "schema_version": "0.2",
  "model": {
    "glb": "building.draco.glb",
    "up_axis": "Y",              // already converted; the viewer never converts
    "unit": "m"
  },
  "eye_height": 1.6,

  "plan_groups": [{
    "id": "g0",
    "representative_floor": 0,   // the storey that was actually rendered
    "floors": [0,1,2,3,4,5,6,7,8,9,10,11],
    "match": "iou",              // "hash" = exact, "iou" = near-match
    "min_iou": 0.9937            // the worst match in the group
  }],

  "floors": [
    { "index": 0, "label": "Floor 1", "z": 3.3036, "plan_group": "g0" }
  ],

  "hotspots": [
    {
      "id": "f0_h0",
      "floor_index": 0,
      "plan_group": "g0",
      "position": [43.2906, 4.9036, -20.688],   // glTF space, Y up
      "label": "Room 1",
      "kind": "room",                            // "room" | "grid"
      "area_m2": 32.0,
      "clearance_m": 1.75,
      "cubemap": {
        "dir": "cubemaps/g0/h0",
        "faces": ["px","nx","py","ny","pz","nz"],
        "ext": "jpg", "size": 1024, "lo": 256
      },
      "shares_cubemap_with": null      // this one was rendered
    },
    {
      "id": "f1_h0",
      "floor_index": 1,
      "position": [43.2906, 7.9035, -20.688],   // same X/Z, one storey up
      "cubemap": { "dir": "cubemaps/g0/h0" },   // SAME directory
      "shares_cubemap_with": "f0_h0"            // reused
    }
  ]
}
```

The two hotspots above are the dedup made visible: same plan, same X/Z, 3 m apart
in Y, **one set of images between them**. 120 hotspots, 10 directories.

### `report.json` — the diagnostic record

From the run currently in `output/`:

```jsonc
{
  "status": "partial",              // exported, but with ≥1 error
  "timings_sec": {
    "intake": 0.55, "measure": 0.48, "floors": 0.18, "plans": 0.21,
    "rooms": 0.04, "cubemaps": 83.27,       // 97% of the run
    "optimize": 0.95, "export": 0.36, "total": 86.07
  },
  "geometry": {
    "before": { "objects": 17, "triangles": 191310, "vertices": 94300 },
    "after":  { "objects": 17, "triangles": 78348,  "vertices": 37984 },
    "target_ratio": 0.3,            // what was asked
    "actual_keep_ratio": 0.4095,    // what was achievable
    "actual_reduction_pct": 59.05
  },
  "cubemaps": {
    "rendered": 10, "reused": 110,  // the 12× saving
    "engine": "BLENDER_EEVEE", "size": 1024, "exposure": -0.4
  },
  "classification": {
    "solver_pressure": 0.0,         // pinned at the floors
    "floor_limited": true,          // and honest about it
    "predicted_keep": 0.3995,
    "by_class": {
      "thin_feature": { "objects": 4, "triangles": 158874, "ratio": 0.45, "share_pct": 83.0 },
      "solid":        { "objects": 7, "triangles": 31560,  "ratio": 0.15, "share_pct": 16.5 },
      "slab":         { "objects": 2, "triangles": 712,    "ratio": 0.05, "share_pct": 0.4 },
      "trivial":      { "objects": 4, "triangles": 164,    "ratio": 1.0,  "share_pct": 0.1 }
    }
  },
  "diagnostics_summary": { "info": 11, "warning": 5, "error": 1 }
}
```

That `share_pct: 83.0` on `thin_feature` is the whole story of why the target was
missed: 83% of this building is geometry that shatters under collapse.

Diagnostics raised on this run:

| Code | Count | Meaning |
|---|---|---|
| `LOOSE_VERTICES` | 8 | the weld repair working |
| `DEGENERATE_FACES` | 2 | zero-area faces removed |
| `NO_UV` | 2 | unwrapped with Smart UV Project |
| `NO_MATERIAL` | 1 | substituted default grey |
| `NO_ROOMS_DETECTED_USING_GRID` | 1 | undivided shell, fell back to grid |
| `AUTO_LIGHTING_ADDED` | 1 | scene had no lights |
| `TRIANGLE_BUDGET_MISSED` | 1 | floor-limited at 41% |
| `UV_DISTORTED` | 1 | **the one error** — `Box002` UV area moved 0.33× |

---

## 21. A full run, traced

`uv run p02 intake_model/<project>/source/model.fbx --serve`

| # | Where | What happens |
|---|---|---|
| 1 | `cli.py` | Validate ratio and extension; find Blender 5.1.2. |
| 2 | `cli.py` | Launch `blender --background --factory-startup --python process_scene.py -- …` |
| 3 | `process_scene` | Add `src/` and `blender/` to `sys.path`; import the world. |
| 4 | `bl_intake` | Import FBX. 17 meshes, 14 materials, 8 embedded textures. |
| 5 | `bl_intake` | **Weld.** 347,616 → 94,300 vertices. `LOOSE_VERTICES` ×8. |
| 6 | `bl_intake` | Relink textures (all packed, nothing to do); fix colour spaces by wiring; backstop materials. |
| 7 | `bl_geometry` | One pass over 191,310 triangles → world-space faces, wall spans, island stats. |
| 8 | `floors` | Histogram Z. 14 levels → site at 0.02 excluded, roof at 39.77 excluded, **12 storeys** at exactly 3.00 m spacing. |
| 9 | `bl_geometry` + `rooms` | For each storey: slice at z+1.6, rasterize, dilate ×1. |
| 10 | `signature` | Hash each plan; all 12 match at **IoU 0.9937** → **1 plan group**. |
| 11 | `rooms` | On floor 0: 2 lightwell shafts qualify as rooms, 4 large volumes get grid samples → **10 capture points**. `NO_ROOMS_DETECTED_USING_GRID`. |
| 12 | `hotspots` | **10 render jobs**, **120 hotspots** (10 × 12 floors). |
| 13 | `process_scene` | Ray-fan clearance check on all 10 jobs. |
| 14 | `bl_render` | No lights found → add Nishita sky + sun. Render 10 × 6 = 60 faces at 1024², plus 60 previews. **83 s.** |
| 15 | `classify` | 4 thin_feature (83%), 7 solid, 2 slab, 4 trivial. Solver pins at floors → `floor_limited`. |
| 16 | `bl_optimize` | Dissolve then collapse per class. `Cone001` 115,872 → 37,536 by dissolve alone. |
| 17 | `bl_optimize` | UV area check per object. `Box002` at 0.33× → `UV_DISTORTED`. |
| 18 | `bl_optimize` | Export `building.glb` (17.8 MB) and `building.draco.glb` (15.1 MB). |
| 19 | `process_scene` | Write `hotspots.json` and `report.json`. Status `partial`. |
| 20 | `cli.py` | Filter Blender's output to the summary lines; serve on :8000. |
| 21 | `app.js` | Fetch `hotspots.json`, load the Draco GLB, build 120 sprites, build the floor strip, show floor 1. |
| 22 | browser | Click a marker → fade → cubemap loads 256 px then 1024 px → look around → Esc to return. |

---

## 22. What it deliberately does not do

| Gap | Why |
|---|---|
| **No texture optimization** | No ORM packing, no resize, no KTX2. This is where the file size is: 15 MB Draco GLB, ~13 MB of it eight embedded 2K JPEGs. Draco saved only 11% because it compresses geometry, not textures — **direct evidence for `CLAUDE.md` §4's "KTX2 is mandatory, not an optimization"**. |
| **No fixture with real interiors** | Room detection is built and unit-tested against synthetic grids but has never run on a model containing actual rooms. The test building's only enclosed spaces are two lightwell shafts. **Top blocker**, mirroring `CLAUDE.md` §7. |
| **Cycles path unexercised** | `--engine cycles` is wired but only EEVEE has been run end to end. |
| **Viewer verified by construction, not by eye** | Assets resolve, modules parse — but panorama seams and marker placement need a human in a browser. |
| **`gltfpack` not used** | `CLAUDE.md` §6 wants it pinned; it isn't installed, so Blender's built-in Draco stands in. |

### Documented deviations from `CLAUDE.md`

| Rule | What P0.2 does | Why |
|---|---|---|
| §3: panoramas rendered in Corona, never Blender | Auto-renders in Blender | A prototype needs imagery. `hotspots.json` points at a *directory* of face images; Corona renders can replace them with no schema change. |
| §0: `viewer/` deferred | Builds `website/` inside p0.2 | Prototype viewer, not the product viewer. |
| §4: UV transfer via `TOPOLOGY` | `POLYINTERP_NEAREST`, recovery only | Topology mapping needs face-for-face correspondence, which decimation has just destroyed — [see the UV story](#uvstory). |

### The one thing to take away

P0.2's most valuable output is not the GLB or the cubemaps. It is a set of
**measured facts that contradict the plan**:

1. **Welding is mandatory and undocumented.** 3ds Max FBX arrives as loose
   triangles; nothing downstream works until it's welded.
2. **Name-based classification caught 0%.** Geometry heuristics carried all of it.
3. **Filename-based colour-space detection corrupts real models.** Five of eight
   textures were named as data maps and wired into Base Color.
4. **The prescribed UV transfer destroys UVs on tiled materials.** Measured at
   +21,311% UV area.
5. **A 70% cut is not always reachable**, and saying so beats faking it.
6. **Textures, not geometry, are the file size.**

Each of those is a line in `CLAUDE.md` that needs revising, backed by a number
rather than an opinion.
