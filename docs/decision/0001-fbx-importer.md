# ADR-0001 — Prefer `wm.fbx_import` over the legacy FBX importer

**Status:** Accepted
**Date:** 2026-08-08

## Context

Blender 5.x ships **two** FBX importers:

- `bpy.ops.import_scene.fbx` — the legacy Python importer
- `bpy.ops.wm.fbx_import` — the newer C++ importer, and what File → Import uses

They are not interchangeable. Measured A/B on the same file, same Blender 5.1.2:

| Importer | Metallic | Geometry |
|---|---|---|
| `import_scene.fbx` (legacy) | **1.0** | 192,960 tris, 8 textured base colours |
| `wm.fbx_import` (C++) | **0.0** | 192,960 tris, 8 textured base colours |

Identical in every respect except a fabricated material value.

The legacy importer produced byte-identical
`(metallic 1.0, roughness 0.859, specular 0.0)` across **all 15 materials** —
including "Wall Paint", "Ceramic" and "Solid Glass". Fifteen materials sharing one
identical triple is the signature of an importer default, not artist intent.

### Why it was invisible for so long

A fully metallic surface has no diffuse response — it shows only reflections. In
Blender's viewport, which ships a studio HDRI, this reads as plausible brushed
metal. Exported to glTF and viewed in a browser with a modest environment, the
same material has nothing to reflect and the building renders near-black.

The bug therefore presents as "looks fine in Blender, broken on the web", which
sends you looking at the exporter, the viewer, the tone mapping and the
environment — every part of the system except the importer.

It also exported *faithfully*: `metallicFactor` was simply omitted, and the glTF
default is `1.0`. The written file was a correct encoding of corrupt input.

## Decision

Use `wm.fbx_import`. Fall back to `import_scene.fbx` only when it raises, and
record `FBX_IMPORTER_FALLBACK` when that happens.

Because the legacy path survives as a fallback, keep a narrow safety net: if every
material shares one identical `(metallic, roughness, specular)` triple **and**
metallic is 1.0, treat it as the legacy importer's signature, force metallic to
0.0, and log `MATERIAL_IMPORT_ARTIFACT`.

Additionally, **always write `metallicFactor` explicitly** on export, so the glTF
default can never silently reassert itself. The export gate enforces this.

## Consequences

- Both importers must remain reachable; agency FBX files span 2018–2024 exporters
  and the two fail on *different* malformations, so the fallback is real coverage.
- The uniformity heuristic is a heuristic. A scene where every material genuinely
  is the same metal would be misread. Acceptable: that scene does not occur in
  arch-viz, and the alternative is shipping black buildings.
- Material `type` tags from `metadata.json` remain the right long-term source for
  genuinely metallic materials.

## Alternatives considered

**Guess per material from its name.** "13 - Brushed Metal #2" is clearly metal.
Rejected — this is the identical mistake ADR-0003 exists to prevent, and the same
asset proves it: five textures named `*_AmbientOcclusion` / `*_METALNESS` are all
wired into Base Color. Names in this domain are unreliable.

**Fix it in the viewer.** Override metallic at load. Rejected — it corrupts the
delivered GLB for every other consumer and hides the defect rather than removing
it.

**Improve the viewer environment instead.** Metal needs something to reflect, so a
better environment does make it look less broken. Rejected as *the* fix — it
papers over wrong data. (A better environment was adopted separately, for its own
reasons.)

## Notes

An earlier prototype (`p0.1`) already preferred `wm.fbx_import` with a fallback.
The later, more sophisticated prototype dropped it and called the legacy importer
directly. **This was a regression in a rewrite**, which is precisely the kind of
loss an ADR is meant to prevent.
