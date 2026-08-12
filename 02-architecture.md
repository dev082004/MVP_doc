# 02 — Architecture

## The constraint that shapes everything

**Blender ships its own Python interpreter, and it does not see the project's
virtual environment.**

```
uv environment                    Blender's bundled Python
CPython 3.13.13                   CPython 3.13.9
.venv/bin/python3                 Blender.app/.../python/bin/python3.13
bpy: NOT importable               bpy: importable
```

Packages *can* be installed into Blender's interpreter
(`uv pip install <pkg> --python <blender-python>`, see `CLAUDE.md` §6), so this is
a path problem rather than a wall. But the two interpreters are separate
processes with separate module paths, and that dictates the layout.

## Three-layer split

```
  LAYER 1 — orchestration (uv env)        LAYER 2 — Blender (bpy)
  ────────────────────────────────        ───────────────────────
  CLI: arg parsing, Blender               import / repair / measure
  discovery, subprocess launch,           render / optimize / export
  exit codes
        │                                          │
        └──────── subprocess ──────────────────────┘
                            │
                  LAYER 3 — shared pure modules
                  stdlib-only, imported by BOTH
                  (detection maths, report model, coordinate conversion)
                            │
                            ▼
                  files on disk = the only channel
                  report.json · *.glb · hotspots.json
```

### Why the shared layer is stdlib-only

It is imported by both interpreters. A third-party import there fails at runtime
inside Blender while every test still passes, because the tests run in the uv
environment where the package exists.

This is a **preference with a reason, not an absolute**: keeping detection maths
dependency-free is what allows 146 unit tests to run in 0.08s with no Blender at
all. When a dependency genuinely earns its place (Pillow for texture packing),
install it into both interpreters and record it in the image.

### The analysis seam

Only one Blender module touches `bpy` on the analysis path. It measures the scene
into plain tuples and hands them to the pure layer:

```
bl_geometry.py  ──emits──▶  HorizontalFace, WallSpan, Segment, ObjectMetrics
   (bpy)                            (plain NamedTuples)
                                          │
                                          ▼
                       floors.py · rooms.py · signature.py · classify.py
                                    (no bpy anywhere)
```

That is why floor detection, flood fill, plan matching and the ratio solver are
all unit-testable against hand-built fixtures.

## Stage order

Fixed. Each step depends on the last.

```
 1  import            FBX/GLB → Blender scene
 2  repair            weld, degenerate faces, loose verts, UVs, materials
 3  measure           world-space faces, wall spans, per-object metrics
 4  detect floors     Z-histogram → storeys; filter site plane and roof
 5  slice + group     per-storey wall raster → plan signature → group identical
 6  capture points    flood fill → rooms; grid-sample undivided volumes
 7  render cubemaps   one per capture point per PLAN GROUP
 8  optimize          decimation (opt-in), textures
 9  export            GLB, then validate the written file
10  report            report.json + hotspots.json
```

### Two orderings are load-bearing

**Cubemaps render before optimization.** The panoramas are the photographic
record of the scene; shooting them through a model that has already lost
triangles throws away the thing you came for.

**Plans are grouped before rendering.** A tower repeats one plan a dozen times.
Detecting that turns 120 renders into 10 — measured, and rendering is 97% of
runtime. Skipping it multiplies the most expensive stage by 12.

## World-space discipline

Blender's `polygon.area` and `polygon.normal` are **local-space** and silently
wrong for any object carrying a scale — which, in a 3ds Max export, is every
object. All measurement transforms to world space first, and normals use the
inverse transpose of the object matrix (normals do not transform like positions
under non-uniform scale).

## Coordinate conversion

Blender is **Z-up**; glTF and three.js are **Y-up**.

```
blender_to_gltf: (x, y, z) → (x, z, -y)
gltf_to_blender: (x, y, z) → (x, -z, y)
```

This lives in exactly one module and is called from everywhere that crosses the
boundary. Doing it inline elsewhere is how markers end up floating in plausible
but wrong places with no error message.

**A conversion module only rotates — it cannot absorb a scale error.** Scale
agreement between the Blender scene and the exported file is asserted separately
by the export gate.

## Failure philosophy

Every stage is wrapped so that a failure downgrades status and continues. The
report is written **outside** the top-level try/except, so a valid `report.json`
exists no matter what explodes. It is the only artifact the caller is guaranteed.

## Prototype provenance

| Prototype | Proved |
|---|---|
| `blender_core_processing_p0.1/` | The three-layer split; browser measurement of load/size/FPS |
| `blender_core_processing_p0.2_updated/` | Geometry-only floor and room detection; plan dedup; material fidelity; export validation |

Neither is `pipeline/`. `CLAUDE.md` §0 is explicit that the real worker starts
fresh — these exist to produce measurements, and they have.
