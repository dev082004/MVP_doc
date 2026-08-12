# 03 — Stage Specifications

Each stage states its purpose, inputs, outputs, tuning constants and failure
modes. **No stage may raise past its own boundary** — failures become diagnostics.

Constants marked **measured** were derived from a real asset. Constants marked
*estimate* come from the research documents and are unvalidated.

---

## 1. Import

**Purpose** — get the intake file into a Blender scene without losing material or
normal data.

| | |
|---|---|
| Input | `.fbx` (Track 1), `.glb`/`.gltf` (Track 2, Phase 2), `.blend`, `.obj` (last resort) |
| Output | Populated Blender scene |
| Failure | `EMPTY_SCENE` if no meshes; fatal if the format is unreadable |

**The importer choice is not cosmetic.** Blender 5.x ships two FBX importers and
they produce *different material data* from the same file. Use `wm.fbx_import`
(C++), falling back to `import_scene.fbx` only on failure and recording
`FBX_IMPORTER_FALLBACK` when it happens. See
[ADR-0001](decisions/0001-fbx-importer.md).

Start from a genuinely empty scene, or Blender's default cube, camera and light
are exported with the model.

---

## 2. Repair

**Purpose** — make the geometry usable. This is the highest-value stage in the
pipeline and the least obvious.

### 2a. Weld — mandatory, not cleanup

A 3ds Max FBX arrives with meshes **exploded into disconnected triangles**. Until
welded, collapse decimation has no shared edges to collapse and every
island-based heuristic reads garbage.

| Constant | Value | Basis |
|---|---|---|
| Merge distance | `1e-4` | **measured** — tight enough to keep genuinely separate parts apart, loose enough to close FBX float noise |

Also removes degenerate (zero-area) faces and loose vertices.

**Custom split normals survive the weld** — verified, `has_custom_normals` is
`True` before and after. The weld is not responsible for shading artifacts. See
[ADR-0002](decisions/0002-weld-first.md).

### 2b. Texture relink

Broken texture paths affect **30%+ of real submissions** (*estimate*, from intake
research). Index candidate files by **stem, case-insensitive, any extension** —
agencies routinely re-save `.jpg` as `.jpeg`.

Textures embedded in the FBX arrive already packed and need nothing.

### 2c. Colour space

Blender's FBX importer marks every texture sRGB. Roughness and metalness maps read
through an sRGB curve give the wrong surface response everywhere.

**Decide from how the texture is wired, never from its filename.** See
[ADR-0003](decisions/0003-colorspace-from-wiring.md).

### 2d. Material backstop

Any mesh with no material gets a shared default grey plus a diagnostic. A degraded
result beats a refusal to run.

### 2e. UVs

**Every mesh must leave intake with a UV layer.** A mesh without UVs cannot be
textured regardless of material quality — and this is true whether or not anything
later reduces its triangle count, so it belongs here rather than in the optimizer.

| Constant | Value | Basis |
|---|---|---|
| Smart UV Project angle limit | 66° | *estimate* — Blender's default over-fragments arch-viz geometry |
| Island margin | 0.02 | *estimate* |

---

## 3. Measure

**Purpose** — walk every mesh **once** and cache what the analysis stages need.

Emits plain data: upward-facing polygons (world Z, world area, XY extent),
near-vertical polygon spans, per-object metrics, per-object bounding boxes.

| Constant | Value | Meaning |
|---|---|---|
| Up-normal minimum | 0.9 | floor-like above this |
| Wall-normal maximum | 0.35 | wall-like below this |

The dead band between them is soffits, ramps and chamfers — **deliberately counted
as neither**, rather than forced into a category where they pollute the histogram.

Caching matters: the room slice runs once per storey, and re-deriving world
coordinates a dozen times over ~190,000 polygons is the difference between
seconds and minutes.

---

## 4. Floor detection

**Purpose** — find storey levels from geometry alone. Object names in real agency
scenes are 3ds Max defaults and carry no information.

**Method** — histogram upward-facing *world* area against Z; storeys appear as
peaks. Group above-threshold bins into runs rather than hunting local maxima: a
slab split across two bins is one level, and run-grouping gets that right with no
peak-prominence tuning.

| Constant | Value | Basis |
|---|---|---|
| Bin size | 0.1 m | **measured** |
| Run gap tolerance | 0.3 m | **measured** |
| Minimum storey height | 2.0 m | *estimate* — merges slab top with finished floor |
| Site extent factor | 1.5× median | **measured** |
| Roof wall fraction | 0.45 | **measured** — storeys enclosed ~2,297 units of wall, the roof only 720 (31%) |

**Two things masquerade as floors and must be filtered:**

- **Site plane** — the ground mesh is a large flat upward surface. Caught by XY
  extent: much wider than the building footprint.
- **Roof** — a real slab, distinguished only by having no walls enclosing anything
  above it. Demote from the top down and stop at the first level that does have
  walls above — a mid-building level with little wall above it is an **atrium**,
  not a roof.

Level height is the **area-weighted centroid** of its faces, not the bin centre.

---

## 5. Plan grouping

**Purpose** — recognise that many storeys share one floor plan. This is where the
render budget is won.

**Method** — rasterize each storey's wall slice, re-base the occupied cells on
their own bounding box so position does not matter, hash them, then compare by
intersection-over-union.

| Constant | Value | Basis |
|---|---|---|
| IoU threshold | 0.98 | **measured** — real floors matched at 0.9937; below this you start merging a 2BHK with a 3BHK and reusing the wrong cubemaps |

Hash first (nearly free, catches exact matches), IoU second (real geometry has
float noise and the occasional extra bollard). Cluster greedily against group
representatives: O(floors × groups), and membership stays deterministic — a floor
always joins the earliest group it matches, so the representative is the lowest
storey with that plan.

---

## 6. Capture points

**Purpose** — find where to stand on each distinct plan.

**Method** — slice at eye height, rasterize wall crossings, flood-fill from
outside, and treat each unreachable component as enclosed.

| Constant | Value | Basis |
|---|---|---|
| Cell size | 0.25 m | **measured** |
| Dilation | 1 cell, 8-connected | **measured** |
| Flood fill | 4-connected | **measured** |
| Room area range | 4–60 m² | *estimate* |
| Minimum room dimension | 1.2 m | *estimate* |
| Minimum clearance | 0.4 m | *estimate* |
| Grid spacing (fallback) | 8 m | *estimate* |
| Eye height | 1.6 m | *estimate*, configurable |

**Three details do most of the work:**

**Slice, don't project.** Compute the true plane–polygon intersection. Projecting a
wall's outline downward paints its whole height into the raster and closes off
doorways the slice passes cleanly through.

**Dilate before filling.** Coincident facade faces leave hairline gaps that leak
the exterior fill inside and merge every room into the outdoors. Dilate 8-connected
to close gaps; fill 4-connected so nothing squeezes diagonally between two wall
cells touching at a corner. The pair works together.

**Use the pole of inaccessibility, not the centroid.** An L-shaped room's centroid
falls in the notch — inside a wall — and the camera then renders the inside of a
brick. The pole (furthest free cell from any wall) is always inside, and is also
the most open place to stand. Break ties toward the centroid so a symmetric room
gets its middle rather than an arbitrary corner of the plateau.

**Rooms *plus* a grid, not either.** Finding one room does not mean the floor is
covered. On a real exterior asset the only room-sized enclosed spaces were two
lightwell shafts; the genuine volumes were 150, 146, 78 and 76 m² of undivided
shell. Returning only the shafts would leave a 26×61 m floor plate with two
capture points, both in service voids.

---

## 7. Cubemap render

**Purpose** — a 360° capture at each capture point. **Prototype-only by design** —
production panoramas come from Corona on the agency's machine.

One render job per capture point **per plan group**; one hotspot per capture point
**per floor**. That asymmetry is the dedup.

| Constant | Value | Note |
|---|---|---|
| Field of view | 90° | not a choice — it is what makes six faces tile seamlessly |
| Face order | `px nx py ny pz nz` | must match three.js `CubeTextureLoader` |
| Face size / preview | 1024 / 256 px | *estimate*; preview is downscaled, never re-rendered |

The face-orientation table is the **highest-risk constant in the pipeline**: a
wrong entry does not raise, it produces six plausible images that tile into a
broken panorama. It requires a dedicated self-test that renders a room with six
known-coloured walls and asserts each face shows the right one.

Scenes routinely arrive with **no lights at all**; rendering as-is gives six black
images. Add a sky and sun when none exist, and record it.

---

## 8. Optimize

### 8a. Planar dissolve — repair, not reduction

Merges coplanar faces without moving a vertex. **Geometrically lossless.**
Measured: 191,310 → 107,956 triangles scene-wide.

Because it removes redundancy rather than detail, it belongs on the default path.

### 8b. Decimation — opt-in

Moves geometry, and therefore costs UV fidelity. **Off by default** while material
and texture parity is the priority. See
[ADR-0005](decisions/0005-decimation-opt-in.md).

Per-class ratios solved against a triangle target, with per-class floors and a
tolerance governing how fast the solver may push each class toward its floor — so
slabs are spent before railings are touched. Thin features use **planar dissolve,
not collapse**; collapse shatters them.

**The target is a request, not a guarantee.** When a scene floors out, say so and
record why rather than wrecking the model to hit a number.

### 8c. Textures — NOT BUILT

The largest remaining gap. Textures are **82% of the exported file**. Required:
resize to budget, ORM channel packing, KTX2/Basis. Without KTX2, twenty materials
of uncompressed 1024² RGBA is roughly 400 MB of GPU memory and mid-range phones
run out.

---

## 9. Export and validation

Write GLB, then **read the written file back and assert on it**.

| Export flag | Why |
|---|---|
| `export_apply` | bake remaining modifiers into the mesh |
| `export_yup` | Blender is Z-up, glTF is Y-up; omit it and the model lies on its side with no error |

Gate checks: world bbox agrees with the Blender scene, every material states
`metallicFactor` explicitly, every image resolves, every primitive carries
`TEXCOORD_0`, triangle count matches, file size within budget.

**The bbox check must walk the node hierarchy.** glTF accessor `min`/`max` are in
mesh-local space; node transforms carry the unit conversion. Reading accessors
alone reports a wildly wrong size and sends you hunting a scale bug that does not
exist — that happened during development and cost real time.
