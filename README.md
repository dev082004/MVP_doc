# P0.2 — floor/room detection, cubemap hotspots, optimized GLB + viewer

Takes an FBX building, works out where its storeys and rooms are **without being
told**, renders a 360° cubemap from eye height at each capture point, optimizes
the geometry, and serves a three.js viewer where every capture point is a
clickable marker on the building.

P0.1 answered "how fast is it". P0.2 answers "is it a walkthrough".

**This is a prototype instrument, not the product pipeline.** `pipeline/` and
`viewer/` remain unscaffolded — see `CLAUDE.md` §0.

## Requirements

- Blender **5.1.2** (verified). Set `BLENDER_BIN`, or pass `--blender`.
- [uv](https://docs.astral.sh/uv/)
- No network at runtime. three.js r180 is vendored under `website/vendor/`.

## Usage

```bash
uv run p02 intake_model/<project>/source/model.fbx --serve

uv run p02 model.fbx --skip-cubemaps     # geometry only; iterate on detection
uv run p02 model.fbx --engine cycles --samples 256
uv run p02 model.fbx --target-ratio 0.4  # fraction KEPT
uv run p02 --selftest-cubemap            # verify cube face orientation
uv run p02 --serve-only                  # serve an existing output/
```

Outputs land in `output/`:

```
output/
├── building.glb              uncompressed
├── building.draco.glb        Draco-compressed
├── hotspots.json             the pipeline → viewer contract
├── report.json               floors, plans, per-object decimation, diagnostics
└── cubemaps/<group>/h<n>/    px|nx|py|ny|pz|nz .jpg (1024) + .lo.jpg (256)
```

Exit codes: `0` success, `1` processing failed, `2` bad arguments.

### The viewer must be served over HTTP

`file://` blocks the `fetch` of `hotspots.json` and the GLB. Use `--serve`, or
`python3 -m http.server` **from this directory** — the page loads from
`../output/`, so both directories must sit under one document root.

## How it works

Stage order is fixed, and each step depends on the last:

```
import → repair(weld) → measure → detect floors → slice each floor
       → group identical plans → find capture points → render cubemaps
       → classify → decimate → export GLB → write hotspots.json
```

Two orderings are load-bearing:

- **Cubemaps render before decimation.** They are the photographic record of the
  scene; shooting them through a model that has lost 60% of its triangles is
  throwing away the thing you came for.
- **Plans are grouped before rendering.** A tower repeats one plan a dozen times.
  Detecting that turns 120 renders into 10.

### Floor detection

Histogram upward-facing face **world** area against Z. Storeys fall out as peaks.
Two things masquerade as floors and are filtered: the **site plane** (much wider
XY extent than the building) and the **roof** (a real slab, but with no walls
enclosing anything above it).

### Room detection

Slice the building at eye height, rasterize every wall crossing the plane,
flood-fill the free space, and call each enclosed component a room. Two details
do most of the work:

- **Dilate before flood-filling.** Coincident facade faces leave hairline gaps
  that leak the fill outside and merge every room into the exterior.
- **The centre point is the pole of inaccessibility, not the centroid.** An
  L-shaped room's centroid lands inside a wall, and the camera then renders the
  inside of a brick.

Rooms **plus** a viewpoint grid through any enclosed volume too large to be a
room — both, not either. Finding one room does not mean the floor is covered.

### Decimation

Per-class ratios (`CLAUDE.md` §4), solved against a triangle target. Classes
have a **floor** they will not go below, and a **tolerance** governing how fast
the solver may push them there, so slabs are spent before railings are touched.
`thin_feature` uses planar dissolve, not collapse.

## Measured results — `residential-complex-modern-apartment-building`

Blender 5.1.2, M-series Mac, EEVEE, 1024² cubemaps. **Total 1m58s.**

| | |
|---|---|
| Input | 25.2 MB FBX, 17 objects, 192,960 tris, 14 materials, 8 embedded textures |
| Floors detected | **12**, spacing exactly 3.00 m, z = 3.30 → 36.30 |
| Also detected | site plane at z=0.02 (excluded), roof at z=39.77 (excluded) |
| Distinct plans | **1**, matched by IoU 0.9937 across all 12 storeys |
| Capture points | 10 per floor → **120 hotspots from 10 cubemap renders** (12× saving) |
| Geometry | 191,310 → **78,348 tris** (59.0% removed) |
| GLB | 16.02 MB plain, 14.23 MB Draco |
| Cubemaps | 3.8 MB for 60 faces + 60 previews |

Stage timings: intake 0.6s, measure 0.5s, floors 0.2s, plans 0.2s, rooms 0.05s,
**cubemaps 100.3s**, optimize 15.3s, export 0.4s.

### What this model taught us

**The FBX arrives exploded into disconnected triangles.** `Cone001` imported as
115,872 triangles across 115,872 separate islands — 347,616 vertices for what is
really 58,752. Until that is welded, decimation cannot work (collapse needs
shared edges) and every island-based heuristic reads garbage. Welding costs
**0.9% of triangles** and fixes both. It is the single most important repair
step, and it is not in `CLAUDE.md`'s pipeline description.

**Planar dissolve massively outperforms its target on thin geometry.**
`Cone001` — 37mm-thin balusters, 60% of the whole scene — went 115,872 → 37,536
(keep 0.32) through *lossless coplanar merging alone*, beating its 0.45 floor
without collapsing a single element:

| object | class | strategy | before | after | keep |
|---|---|---|---|---|---|
| Cone001 | thin_feature | planar | 115,872 | 37,536 | 0.32 |
| Shape | thin_feature | planar+collapse | 32,994 | 21,830 | 0.66 |
| Line003 | solid | collapse | 17,260 | 11,660 | 0.68 |
| Box002 | thin_feature | planar+collapse | 7,704 | 3,466 | 0.45 |
| Line005 | solid | collapse | 7,008 | 1,050 | 0.15 |

**The 30% target is not reachable, and the pipeline says so instead of faking
it.** 83% of this scene is thin louvre and mullion geometry that shatters under
collapse. The solver stops at the class floors, lands at 41% kept, and records
`TRIANGLE_BUDGET_MISSED` with the reason. Further reduction needs element
removal or LODs, not simplification. 78K triangles is comfortably inside
`CLAUDE.md` §4's 80–120K budget — which is the number that actually matters.

**Draco is almost irrelevant here; textures are the whole file.** 16.02 → 14.23 MB
is an 11% saving, because ~13 MB of the GLB is eight embedded 2K JPEGs. This is
direct evidence for `CLAUDE.md` §4's "KTX2/Basis is mandatory, not an
optimization" — and P0.2 does not implement it, so the GLB is over the 5–12 MB
target.

**This model has no interiors.** Its only room-sized enclosed spaces are two
lightwell shafts (32.0 and 28.8 m², both 4.8×11.2 m slots between the wings).
The four genuine volumes are 150, 146, 78 and 76 m² of undivided shell. The room
detector is correct; the fixture has nothing to find. This is why capture points
are rooms *plus* a grid, and it is the top blocker on validating the room path
for real.

## The cubemap orientation self-test

The face-orientation table in `blender/bl_render.py` is the highest-risk
constant in P0.2. A wrong entry does not raise — it produces six plausible
images that tile into a broken panorama.

```bash
uv run p02 --selftest-cubemap
```

Renders a cube room with six known-coloured emissive walls and asserts each face
shows the right one. It is built as a cube rather than six placed planes: a
plane needs an orientation, and getting *that* wrong would produce failures
indistinguishable from the bug the test exists to catch. The wall colours are
assigned through the same `p02.coords` conversion the renderer uses, so a sign
error there surfaces here instead of cancelling out.

It also pins the view transform to Standard — Blender's default AgX desaturates
saturated emission far enough that cyan reads as white and the test cannot tell
a correct face from a wrong one.

## Layout

```
src/p02/            PURE STDLIB — imported by BOTH uv and Blender's interpreter
  report.py         diagnostics, status rollup, report.json
  floors.py         Z-histogram → storey levels, site/roof filtering
  rooms.py          occupancy grid, flood fill, pole of inaccessibility
  signature.py      plan hashing + IoU → plan groups
  classify.py       geometry-heuristic classification + ratio solver
  coords.py         Blender Z-up ↔ glTF Y-up. THE single conversion point.
  hotspots.py       render jobs + hotspots.json
  cli.py            arg parsing, Blender subprocess, --serve
blender/            runs INSIDE Blender (bpy + stdlib only)
  process_scene.py  orchestrator
  bl_intake.py      import, weld repair, texture relink, colorspace fix
  bl_geometry.py    world-space measurement, plane slicing, island stats
  bl_render.py      cubemap capture + orientation self-test
  bl_optimize.py    decimate, UV preservation, GLB export
website/            three.js viewer, fully vendored (works offline)
tests/              uv run pytest — no Blender needed
```

`src/p02/` is dependency-free because Blender's bundled Python cannot see
uv-installed packages. Keeping the detection maths there is what makes it
testable without launching Blender.

## Tests

```bash
uv run pytest        # 146 tests, no Blender required
```

Covers floor peak-finding (site plane, parapet roof, mezzanine merging), room
flood-fill (L-shaped rooms, hairline leaks, slivers, corridors), the pole of
inaccessibility — including a guard asserting a centroid *would* have failed —
plan hashing and IoU, the coordinate round-trip, and the classifier against the
**real measured metrics** of all 17 intake objects, so a threshold change that
would wreck the fixture fails in pytest rather than three minutes into a Blender
run.

Not covered by pytest, by design: anything requiring Blender. Cube orientation is
covered by `--selftest-cubemap`; the viewer needs a browser.

## Deliberate deviations from CLAUDE.md

| Rule | What P0.2 does | Why |
|---|---|---|
| §3: photoreal panoramas are rendered in Corona, never in Blender | Auto-renders cubemaps in Blender | A prototype needs imagery. `hotspots.json` points at a directory of face images; Corona renders can replace them with no schema change. |
| §0: `viewer/` deferred | Builds `website/` inside p0.2 | Prototype viewer, not the product viewer. |
| §4: ORM packing, texture resize, KTX2 mandatory | Not implemented | Deferred, as in P0.1. The 16 MB GLB is the measured cost — see above. |
| §4: UV transfer via `TOPOLOGY` mapping | `POLYINTERP_NEAREST` | Topology mapping needs face-for-face correspondence, which decimation has just destroyed. |
| §6: pin `gltfpack` | Blender's built-in Draco | `gltfpack` is not installed. Same call P0.1 made. |

## Known gaps

- **No fixture with real interiors.** Room detection is built and unit-tested
  against synthetic grids, but has never run on a model that contains rooms.
  Top blocker, mirroring `CLAUDE.md` §7.
- **Textures untouched.** No ORM packing, no resize, no KTX2. This is where the
  file size is.
- **Cycles path unexercised.** `--engine cycles` is wired but only EEVEE has been
  run end to end.
- **The viewer has been verified by construction, not by eye** — all assets
  resolve over HTTP and the modules parse, but the panorama seams and marker
  placement need a human looking at a browser.
