# 07 — Measured Benchmarks

Everything here was measured. Nothing is projected. Where a number is an estimate
it is labelled.

**Caveat that applies to the whole document: this is one asset.** A single
exterior residential tower, run on one machine. Treat these as existence proofs
and orders of magnitude, not as a distribution.

---

## Environment

| | |
|---|---|
| Blender | 5.1.2 (bundled Python 3.13.9) |
| Host | Apple M-series, macOS |
| Render engine | EEVEE |
| Test asset | `residential-complex-modern-apartment-building`, 25.2 MB FBX |

---

## The asset

| | |
|---|---|
| Objects | 17 meshes |
| Triangles | 191,310 after weld (192,960 before) |
| Materials | 14 from FBX + 1 substituted default |
| Textures | 7 embedded, 1K–2K |
| Source units | **inches** — node scale 0.0254 |
| Real dimensions | 49.10 × 44.74 × 88.33 m |
| Object names | 3ds Max defaults (`Box002`, `Cone001`, `Line003`) |
| Lights | **none** |

---

## Stage timings — full run

| Stage | Seconds | Share |
|---|---|---|
| intake | 0.42 | 0.8% |
| measure | 0.47 | 0.9% |
| floors | 0.17 | 0.3% |
| plans | 0.23 | 0.4% |
| rooms | 0.04 | 0.1% |
| **cubemaps** | **51.94** | **97%** |
| export | 0.28 | 0.5% |
| **total** | **53.6** | |

**Rendering dominates everything else by two orders of magnitude.** Every other
optimization is noise against it — which is what makes plan grouping the single
highest-leverage feature in the pipeline.

Geometry-only runs (`--skip-cubemaps`) complete in **~2 seconds**.

---

## Detection results

| Measurement | Result |
|---|---|
| Storeys detected | **12**, spacing exactly 3.00 m, z = 3.30 → 36.30 |
| Site plane | correctly excluded at z = 0.02 |
| Roof | correctly excluded at z = 39.77 |
| Distinct plans | **1** |
| Plan match confidence | **IoU 0.9937** across all 12 storeys |
| Capture points | 10 per plan |
| Hotspots emitted | **120** |
| Cubemaps rendered | **10** |

### The dedup payoff

**120 hotspots from 10 renders — a 12× saving on 97% of the runtime.** Without
plan grouping, the 52-second render becomes roughly 10 minutes.

The floors did **not** hash identically (float noise), so exact matching alone
would have found 12 distinct plans and saved nothing. IoU is what makes it work,
and 0.9937 is comfortably above the 0.98 threshold without being suspiciously
close to 1.0.

---

## The weld

| | Before | After |
|---|---|---|
| `Cone001` islands | 115,872 | — |
| `Cone001` vertices | 347,616 | 58,752 real |
| Scene vertices | 347,616+ | 94,300 |
| Triangle cost | — | **~0.9%** |

**A 3ds Max FBX arrives exploded into disconnected triangles** — every triangle
its own island. Before welding, collapse decimation has nothing to collapse and
every island-based heuristic reads garbage.

Custom split normals **survive** the weld (`has_custom_normals` true before and
after), so the weld is not responsible for shading artifacts.

---

## Triangulation quality

3ds Max exports pre-triangulated, into slivers:

| Object | Faces | Slivers (edge ratio > 20) | Worst ratio |
|---|---|---|---|
| `Cone001` | 115,872 | **97,920 (85%)** | 68:1 |
| `Box002` | 7,704 | 5,136 (67%) | **143:1** |
| `Shape` | 33,280 | 12,096 (36%) | 60:1 |
| `Line003` | 18,624 | 1,032 (5%) | 63:1 |

**Every face is smooth-shaded.** Interpolating a normal along a 68:1 sliver
produces a gradient down its long axis — visible as diagonal streaking across
flat facade panels. This is wrong in Blender too; it is simply invisible in Solid
viewport shading.

### Planar dissolve — lossless

| | Triangles |
|---|---|
| After weld | 191,310 |
| After planar dissolve (5°) | **107,956** |
| `Cone001` alone | 115,872 → 37,536 |

**44% of the scene's triangles were redundant coplanar subdivision.** No vertex
moves. This is repair, not reduction.

---

## Decimation (opt-in)

With a 30%-keep target requested:

| | |
|---|---|
| Achieved | **41% kept** (191,310 → 78,348) |
| Outcome | `TRIANGLE_BUDGET_MISSED`, floor-limited |
| Reason | **83% of the scene is thin geometry** that shatters under collapse |

Per-object, at the class floors:

| Object | Class | Strategy | Before | After | Kept |
|---|---|---|---|---|---|
| `Cone001` | thin_feature | **planar only** | 115,872 | 37,536 | 0.32 |
| `Shape` | thin_feature | planar+collapse | 32,994 | 21,830 | 0.66 |
| `Line003` | solid | collapse | 17,260 | 11,660 | 0.68 |
| `Box002` | thin_feature | planar+collapse | 7,704 | 3,466 | 0.45 |
| `Line005` | solid | collapse | 7,008 | 1,050 | 0.15 |

`Cone001` — 37 mm balusters, 60% of the whole scene — beat its 0.45 floor
**through lossless coplanar merging alone**, without collapsing a single element.

**The target was unreachable and the pipeline said so** rather than destroying the
louvres to hit a number. Further reduction needs element removal or LODs, not
simplification. 78K triangles is comfortably inside the 80–120K mobile budget,
which is the number that actually matters.

---

## File size — the real finding

| Build | GLB | Draco | Image bytes |
|---|---|---|---|
| Decimated, legacy importer | 17.78 MB | 15.10 MB | 14.56 MB (**82%**) |
| Undecimated, correct importer | 24.81 MiB | 17.99 MB | 18.26 MB |

**Textures are the file.** Draco saved 11% on the first build because it
compresses geometry, and geometry was 18% of the bytes.

This is direct evidence for the standing claim that **KTX2/Basis is mandatory, not
an optimization** — and the pipeline does not implement it yet, which is why the
GLB is well over the 5–12 MB target for a 2BHK.

Cubemaps, by contrast, are cheap: **5.2 MB for 60 faces at 1024² plus 60 previews.**

---

## Material import — the importer matters

Same file, same Blender, identical geometry (192,960 tris, 8 textured base
colours) both ways:

| Importer | Metallic |
|---|---|
| `bpy.ops.import_scene.fbx` (legacy Python) | **1.0** |
| `bpy.ops.wm.fbx_import` (new C++) | **0.0** |

All 15 materials came through the legacy importer with byte-identical
`(metallic 1.0, roughness 0.859, specular 0.0)` — including "Wall Paint" and
"Ceramic". Identical values across every material is the signature of an importer
default, not authorship.

A fully metallic surface has no diffuse response; it shows only reflections. It
survives in Blender because the viewport ships a studio HDRI to reflect. In a
browser with a weak environment the building goes black. See
[ADR-0001](decisions/0001-fbx-importer.md).

---

## Texture map types — the asset's own error

Five of the seven embedded images are named `*_AmbientOcclusion`,
`*_Displacement` and `*_METALNESS` — and **every one is wired into Base Color**.
No normal, roughness, ORM or emissive textures exist at all.

This is the artist dragging whatever map looked right into the diffuse slot, which
is exactly the drag-and-drop library-material behaviour the intake research
describes. It is why colour-space decisions must read the **wiring**, not the
filename. See [ADR-0003](decisions/0003-colorspace-from-wiring.md).

---

## P0.1 baseline — synthetic

For reference, from the earlier flat-ratio harness on generated geometry:

| | |
|---|---|
| Suzanne, subdivided ×2 | 15,744 → 4,722 triangles (**70.01%** removed) |
| Plain GLB | 226 KB |
| Draco GLB | 54 KB (**4.2×**) |

The 4.2× Draco ratio is **misleading and does not survive contact with a real
asset**: Suzanne has zero textures. On the real building the same setting saved
11%.

---

## Open measurement gaps

| Gap | Why it matters |
|---|---|
| **No asset with real interiors** | The entire room path is unvalidated on real geometry |
| No V-Ray or Corona source file | Material extraction is untested against the actual target |
| No mobile device numbers | The whole budget is justified by phone performance |
| Cycles path unexercised | Only EEVEE has been run end to end |
| Texture pipeline unbuilt | The largest single lever on file size is unmeasured |
