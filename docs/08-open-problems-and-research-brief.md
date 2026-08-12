# 08 — Open Problems and Research Brief

**Self-contained handoff.** Written to be pasted into a fresh conversation with no
prior context. Everything needed to reason about the open problems is here.

Last updated: 2026-08-10

---

## 1. What is being built

**ArchVisu** — an automated pipeline that converts an arch-viz agency's 3D source
scene (3ds Max + V-Ray/Corona, sometimes SketchUp) into a mobile-optimized,
browser-based 3D walkthrough reachable from a URL and QR code. No app install.

Target market: Indian real estate / arch-viz agencies. Phase 1 is a single
building, single flat.

**Only the transformation pipeline is in scope.** The API, job queue, dashboard,
storage provider, cloud platform and production viewer are all deliberately
undecided and unscoped. The pipeline is a standalone CLI: a file path in, files on
disk out.

### Non-negotiable domain constraints

- **A large share of target agencies run pirated 3ds Max / V-Ray / Corona** in
  mixed versions (Max 2018–2024, V-Ray 3.x–6.x, Corona 6–11+). Agency-side tooling
  must be a plain script file with no license check, no activation server and no
  third-party plugin dependency. Assume broken installs; degrade, never crash.
- **Client/server boundary is absolute.** Anything requiring 3ds Max runs on the
  agency's machine. There is no server-side 3ds Max.
- **Partial success is a first-class state.** If 3 of 50 materials fail, ship the
  47 that worked and record the 3. A tool that refuses to run gets abandoned.
- **No network at runtime.** Vendored assets, local decoders, packages baked into
  the image.

---

## 2. Stack and environment

| | |
|---|---|
| 3D processing | Blender **5.1.2** headless (`--background --factory-startup`), bundled Python 3.13.9 |
| Orchestration | Python 3.13.13 via `uv` (never pip/poetry/conda) |
| Delivery format | GLB (glTF 2.0) |
| Viewer | three.js r180, fully vendored, offline |
| Agency-side | MaxScript (3ds Max) — **not started** |
| Test host | Apple M-series, macOS |

**Structural constraint that shapes the codebase:** Blender ships its own Python
interpreter and cannot see the project's `.venv`. Packages *can* be installed into
it (`uv pip install <pkg> --python <blender-python>`), but the two are separate
processes. So the code is split three ways:

1. **uv layer** — CLI, argument parsing, Blender discovery, subprocess launch
2. **Blender layer** — everything touching `bpy`
3. **Shared pure layer** — stdlib-only modules imported by *both*: detection maths,
   report model, coordinate conversion

Layer 3 being dependency-free is what allows 146 unit tests to run in 0.08 s with
no Blender at all. Communication between layers 1 and 2 is **files on disk only**.

---

## 3. What has been built

Two prototypes. **Neither is the production pipeline** — they are instruments built
to replace estimates with measurements.

### P0.1 — baseline measurement harness

Loads FBX/`.blend`, records defects without aborting, applies a flat decimation
ratio, exports plain + Draco GLB, serves a `<model-viewer>` page reporting load
time, transfer size and frame rate. 15 tests.

### P0.2 / P0.2_updated — detection, capture, fidelity

Detects storeys and rooms **from geometry alone** (object names in real agency
scenes are 3ds Max defaults like `Box002`, `Cone001` and carry zero information),
groups identical floor plans, renders a cubemap per capture point, exports GLB with
a validation gate, and serves a three.js viewer with clickable hotspots. 146 tests.

Stage order (fixed, each depends on the last):

```
import → repair(weld) → measure → detect floors → slice each floor
      → group identical plans → find capture points → render cubemaps
      → [decimate: OPT-IN, currently off] → export GLB → validate → report
```

Two orderings are load-bearing:
- **Cubemaps render before optimization** — they are the photographic record.
- **Plans are grouped before rendering** — turns 120 renders into 10.

---

## 4. Measured results (one real asset)

Test asset: `residential-complex-modern-apartment-building`, 25.2 MB FBX,
downloaded marketplace exterior. **This is the only real asset ever tested.**

| Measurement | Value |
|---|---|
| Objects / triangles | 17 meshes / 191,310 tris (192,960 before weld) |
| Real dimensions | 49.10 × 44.74 × 88.33 m |
| Source units | **inches** (node scale 0.0254) |
| Storeys detected | **12**, spacing exactly 3.00 m, z = 3.30 → 36.30 |
| Site plane / roof | correctly excluded (z = 0.02 and z = 39.77) |
| Distinct plans | **1**, matched at **IoU 0.9937** |
| Hotspots / cubemaps | **120 hotspots from 10 renders** (12× saving) |
| Full run | **53.6 s**, of which cubemaps 51.9 s (**97%**) |
| Geometry-only run | ~2 s |
| GLB (undecimated) | 24.81 MiB; images are **82% of bytes** |
| Tests | 146 + 15, all green |

### Decimation, when enabled

Requested 30% kept, achieved **41%** (191,310 → 78,348). Floor-limited: **83% of
the scene is thin louvre/mullion geometry** that shatters under collapse. The
pipeline reports `TRIANGLE_BUDGET_MISSED` rather than wrecking the model.

### Planar dissolve — lossless

191,310 → **107,956** triangles with no vertex moved. `Cone001` alone:
115,872 → 37,536. **44% of the scene's triangles were redundant coplanar
subdivision.**

---

## 5. Bugs found and fixed — the domain's actual behaviour

These are the most transferable findings. Each cost real debugging time and each
contradicts a reasonable prior assumption.

### 5.1 The legacy FBX importer fabricates `Metallic = 1.0`

Blender 5.x ships two FBX importers. Same file, same Blender:

```
bpy.ops.import_scene.fbx  (legacy Python)  →  Metallic = 1.0
bpy.ops.wm.fbx_import     (new C++)        →  Metallic = 0.0
```

Identical geometry both ways. All 15 materials arrived with byte-identical
`(metallic 1.0, roughness 0.859, specular 0.0)` — including "Wall Paint" and
"Ceramic". A fully metallic surface has **no diffuse response**; it shows only
reflections. Blender's viewport ships a studio HDRI so it reads as plausible
brushed metal; a browser with a modest environment renders it near-black.

Presents as *"looks fine in Blender, broken on the web"*, which sends you after
the exporter, the viewer, tone mapping and the environment — everything except the
importer. It also exported *faithfully*: glTF defaults an omitted `metallicFactor`
to 1.0, so the file was a correct encoding of corrupt input.

### 5.2 3ds Max FBX arrives exploded into disconnected triangles

`Cone001`: **115,872 triangles across 115,872 separate islands**, 347,616 vertices
for a real 58,752. Every triangle is its own island.

Consequences, both silent: collapse decimation has no shared edges to collapse so
it does nothing; and every island-based classification heuristic reads garbage
(tris-per-island is 1 across the whole scene).

Welding at `1e-4` costs ~0.9% of triangles and fixes both. Custom split normals
**survive** the weld (verified). This step appears in none of the research
documents.

### 5.3 Filename-based colour-space detection corrupts real models

Blender's FBX importer marks everything sRGB; data maps need `Non-Color`. The
obvious fix — read the filename — corrupted the diffuse of most of the building.

**Five of seven textures are named `*_AmbientOcclusion`, `*_Displacement`,
`*_METALNESS` — and every one is wired into Base Color.** The artist dragged
whatever map looked right into the diffuse slot. The filename describes what the
map *was*; only the wiring describes what it *is being used as*.

Fix: decide from node wiring, fall back to filename only when nothing conclusive
is wired. Boundary-anchor the regex — unanchored, `ao` matches half of English and
`orm` matches "platform", "normal", "transform".

### 5.4 The prescribed UV-preservation technique destroys UVs

Standard advice is: snapshot the mesh, decimate, project original UVs back with a
Data Transfer modifier. Measured UV-area change, with vs without that pass:

```
Line003    −16.0%  →  +21,311.5%
Cone001     +0.0%  →     +222.0%
Box414     −45.7%  →     +173.0%
```

Nearest-polygon interpolation is incoherent once UVs tile, and this asset tiles
27–55× (`Line003` spans u ∈ [−13.39, 14.39]). Adjacent destination loops sample
different tiles and the mapping explodes. Topology mapping is not an alternative —
it needs face-for-face correspondence that decimation just destroyed.

Counter-measurement: **Blender's Decimate already interpolates UVs adequately** —
0.0% area change on a 115,872-triangle object. The transfer was solving a
non-problem and creating a large one. It is now recovery-only.

### 5.5 Making a stage optional exposed a hidden responsibility

Turning decimation off shipped two meshes with **no UVs at all**, because Smart UV
Project had been implemented inside the optimizer. A mesh without UVs cannot be
textured regardless of material quality. Moved to intake.

### 5.6 The viewer's environment was the fidelity bottleneck

Most of this building is untextured flat grey at roughness 0.859, so **what you
see on it *is* the environment**. The viewer used a 16×256 two-stop canvas gradient
as both backdrop and IBL — which painted soft diagonal bands across the facade and
turned every downward-facing surface near-black. Replaced with three.js
`RoomEnvironment` (~5 KB, procedural, no asset).

### 5.7 A false alarm worth recording

An early check reported the GLB as **75× oversized**, triggering a real
investigation into a scale bug that did not exist. Cause: reading glTF accessor
`min`/`max` **without applying node transforms**. Accessor bounds are mesh-local;
the node TRS carries the unit conversion (0.0254, inches). Any bbox check must
walk the node hierarchy.

### 5.8 Depth precision

`near = 0.05, far = 1148` gives a 22,954:1 ratio — roughly **6 mm of depth
resolution at 90 m**. The facade has coincident faces, so balcony slabs z-fight.
Fixed with `logarithmicDepthBuffer` and `near = 0.1`.

---

## 6. Open limitations — ranked

### CRITICAL

#### L1. No texture pipeline. Textures are 82% of the file.

Nothing resizes, repacks or compresses textures. The GLB is 24.8 MiB against a
5–12 MB target for a 2BHK. Draco saved only 11% because it compresses geometry,
and geometry is 18% of the bytes.

Required: resize to budget (1024 px walls/floors/hero, 512 px small, never reduce
normal maps below 1024), **ORM channel packing** (AO→R, Roughness→G, Metallic→B,
5 maps → 3), and **KTX2/Basis**. Without KTX2, 20 materials of uncompressed 1024²
RGBA is roughly 400 MB of GPU memory and mid-range phones OOM.

**Complication:** `gltfpack`/`gltf-transform` are not installed, and the intended
deployment forbids fetching tools at runtime. Blender has no native KTX2 encoder.

> **Research needed:** how to produce KTX2/Basis in a Blender-plus-Python pipeline.
> Options to evaluate: pinning `gltfpack` or `toktx` as container binaries;
> `KTX-Software` Python bindings; `basis_universal` via CLI; doing it as a
> post-process outside Blender entirely. Also: what UASTC vs ETC1S costs in quality
> and encode time for architectural textures specifically.

#### L2. No test asset with real interiors.

The entire room-detection path — flood fill, pole of inaccessibility, room
size/dimension thresholds, the 4–60 m² range, 1.2 m minimum dimension, 8 m grid
spacing — is validated **only against hand-built synthetic ASCII grids**.

The one real asset is an exterior. Its only room-sized enclosed spaces are two
lightwell shafts (32.0 and 28.8 m²); its four genuine volumes are 150, 146, 78 and
76 m² of undivided shell. The room detector is correct; the fixture has nothing to
find.

> **Research needed:** where to source realistic arch-viz interior models (a 2BHK
> with genuine partitions, doors, furniture). Then: does slicing at 1.6 m through a
> furnished interior produce sane rooms, or do wardrobes and kitchen islands
> fragment the flood fill? Do doorways at head height keep rooms connected in ways
> that merge them?

#### L3. No V-Ray or Corona source file has ever been tested.

The entire material-extraction premise — that 85–90% of agency materials are
bitmaps in slots and a custom MaxScript can copy file paths — is **unvalidated**.
The MaxScript extractor is not started. The one tested asset is a marketplace
model with already-flattened materials.

> **Research needed:** exact MaxScript property names for VRayMtl and
> CoronaPhysicalMtl across versions (`texmap_diffuse`, `texmap_reflectionGlossiness`,
> `texmap_bump` vs Corona's `baseTex`, `roughnessTex`…), and how they differ between
> V-Ray 3.x/5.x/6.x. Public documentation is thin; this may require live
> experimentation.

### HIGH

#### L4. Sliver triangulation plus universal smooth shading

3ds Max exports pre-triangulated into slivers, and marks every face smooth:

| object | faces | slivers (edge ratio >20) | worst |
|---|---|---|---|
| `Cone001` | 115,872 | **97,920 (85%)** | 68:1 |
| `Box002` | 7,704 | 5,136 (67%) | **143:1** |

Interpolating a normal along a 68:1 sliver produces a gradient down its long axis —
visible as diagonal streaking on flat facade panels. **This is wrong in Blender too**;
it is invisible only in Solid viewport shading.

Planned fix (not yet applied): planar dissolve as a repair step + clear custom
split normals + auto-smooth at 30°. A before/after render confirms auto-smooth
removes the gradients.

> **Open question:** is re-triangulation (limited dissolve → BEAUTY triangulate)
> worth doing on top of auto-smooth, or does the shading fix alone suffice? Also:
> does clearing 3ds Max smoothing groups ever lose genuinely-intended curvature?

#### L5. Pano mode does not hide the building — cubemaps are invisible

`enterPano()` sets `scene.background` to the cubemap but leaves the GLB in the
scene. The camera moves inside the real geometry and the cubemap sits permanently
occluded behind it. Cubemaps render correctly (120 face images, 5.2 MB) and are
simply never seen. Known, ~5-line fix, not yet applied.

#### L6. `contracts/` does not exist

`metadata.json` — the agency-export → pipeline contract — is unbuilt. Both the
MaxScript extractor and the pipeline must be written against it, so it should be
locked before either is finished. A draft exists in doc 04.

Resolved design points: emit only maps that exist and record `null` (do **not**
generate flat grey defaults); `type: glass|emissive` drives a Blender preset rather
than a conversion; per-material `extraction_status` so partial success is
representable.

### MEDIUM

#### L7. No mobile device measurements

The entire optimization budget (80–120K triangles, 1024 px textures, KTX2
mandatory) is justified by mid-range phone performance, and **no measurement has
been taken on any phone**. All numbers come from desktop runs and research
estimates.

#### L8. Khronos glTF-Validator not wired

The in-process export gate checks bbox, explicit `metallicFactor`, image
resolution, `TEXCOORD_0` presence, triangle count and size. That covers failure
modes *this pipeline has produced*, which is not the same as covering the spec.
The npm `gltf-validator` package has no CLI entry point; the correct resolution is
a pinned binary in the container image.

#### L9. Draco blocks a serverless viewer

A single-file offline viewer (base64 GLB inlined into HTML, no server) is possible
and desirable for delivery, but `DRACOLoader` fetches its `.wasm` decoder at
runtime. Draco saved only 11% here, so dropping it may be the right trade — but
that interacts with L1, since KTX2 would change the maths entirely.

#### L10. Tone-mapping/exposure parity between cubemaps and 3D view unverified

Cubemaps render in Blender with AgX at exposure −0.4; the viewer now uses AgX too,
but the two have never been compared side by side. Also, `--engine cycles` is
wired but only EEVEE has been run end to end.

#### L11. Decimation cannot reach target on thin-feature-heavy scenes

83% thin geometry floors the solver at 41% kept. Further reduction needs **element
removal or LODs, not simplification** — out of current scope.

> **Research needed:** what does credible LOD generation look like for repeated
> architectural elements (a balustrade of 200 identical balusters)? Instancing?
> Billboard replacement at distance? Does glTF `EXT_mesh_gpu_instancing` help here?

### LOW / OPERATIONAL

- **Chrome MCP unreliable** — browser automation timed out on every attempt across
  a long session, so all viewer verification has been manual.
- **Blender MCP requires a running GUI addon** — headless fallback gives identical
  numeric results but no screenshots.
- **One machine, one OS.** Everything measured on an Apple M-series Mac.

---

## 7. Research questions, prioritized

1. **KTX2/Basis without `gltfpack`** — how to encode inside a Blender/Python
   pipeline with pinned, offline-capable tooling. Highest impact on file size.
2. **Where to obtain a realistic interior arch-viz fixture**, and what breaks in
   flood-fill room detection when furniture is present.
3. **MaxScript property names for VRayMtl / CoronaPhysicalMtl across versions**,
   and version-detection strategy for a script that must not crash on any of them.
4. **ORM packing inside Blender** — Pillow is the obvious route; what does it cost
   in pipeline time, and how does it interact with textures the artist has wired
   into the wrong slots (see 5.3)?
5. **LOD / instancing for repeated thin elements** as an alternative to
   simplification.
6. **Mobile GPU texture-memory budgets** on the actual target hardware
   (Snapdragon 6-series class, 4 GB RAM) — is 1024 px the right cap?
7. **Does triangulation quality independently affect texture appearance**, or is
   the shading normal the whole story?

---

## 8. Things NOT to re-litigate

Settled by measurement or explicit decision. Reopening them wastes time.

| Settled | Why |
|---|---|
| Weld before anything else | Non-negotiable; nothing downstream works without it |
| `wm.fbx_import` over legacy | Measured metallic corruption |
| Colour space from wiring, not filename | Measured corruption of 5 of 7 textures |
| No unconditional UV transfer | Measured +21,311% UV area |
| Decimation opt-in for now | Fidelity before reduction, until textures are handled |
| No third-party paid dependency agency-side | Cracked-software constraint |
| GLB (not FBX) from SketchUp | FBX from SketchUp loses hierarchy and materials |
| No USD/USDZ in v1 | Blender's USD importer drops MaterialX |
| Emit only textures that exist, mark nulls | Do not ship flat grey PNGs over Indian upload bandwidth |
| Cubemaps are a separate production track | Corona renders replace prototype output with no schema change |
