# ADR-0002 — Weld before anything else touches the geometry

**Status:** Accepted
**Date:** 2026-08-08

## Context

A 3ds Max FBX export arrives with its meshes **exploded into disconnected
triangles**. Measured on the intake asset:

| | |
|---|---|
| `Cone001` triangles | 115,872 |
| `Cone001` separate islands | **115,872** — one per triangle |
| `Cone001` vertices | 347,616 for a real 58,752 |

Nothing is joined to anything. Every triangle is its own island with its own three
unshared vertices.

Two consequences, both fatal and both silent:

1. **Collapse decimation does nothing useful.** Collapse works by merging vertices
   across shared edges. With no shared edges there is nothing to collapse — the
   modifier runs, reports success, and the triangle count barely moves.
2. **Every island-based heuristic reads garbage.** Triangles-per-island is `1` for
   the entire scene, so any classifier that distinguishes a railing from a wall by
   island structure sees one undifferentiated mass.

The second is the more dangerous. It produces confident, wrong classifications
rather than an obvious failure.

### How it was found

Decimation appeared to be working — it ran without error, and the pipeline
reported a reduction. The real cause surfaced only when the numbers were checked
against the mesh, several stages downstream of the actual problem.

## Decision

Weld immediately after import, before measurement, classification or any
optimization.

| Parameter | Value |
|---|---|
| Merge distance | `1e-4` |

Also remove degenerate (zero-area) faces and vertices belonging to no edge.

Weld **per object**, each in its own error handler, so one pathological mesh
cannot take the rest down.

Any metric that depends on island structure — loose-part count, per-island
thickness — must be measured **after** this step and is meaningless before it.

## Consequences

- Costs roughly **0.9% of triangles**. Cheap for what it enables.
- **Custom split normals survive** — verified, `has_custom_normals` is `True`
  before and after, with smooth-face counts preserved. The weld is *not*
  responsible for shading artifacts, which was a suspected cause and is now ruled
  out.
- `1e-4` (0.1 mm) is tight enough to leave genuinely separate parts separate and
  loose enough to close FBX float noise. It has not been tested against a
  millimetre-scale asset such as jewellery or hardware detail.
- This is **not** a cleanup nicety and must never be made optional or moved later
  in the order.

## Alternatives considered

**Merge by distance only at export.** Rejected — classification and decimation
both run before export and both need welded geometry.

**Rely on the FBX exporter to weld.** Not available. It is the agency's 3ds Max
doing the exporting, on a version we do not control, frequently a cracked install.
Assume the worst input.

**Detect and skip when already welded.** Considered and rejected as premature. The
operation is cheap and idempotent; a conditional adds a branch and a way to be
wrong for no measurable saving.

## Notes

This step does not appear in any of the research documents describing the
pipeline. It was discovered by measurement, and without it several later stages
silently produce plausible nonsense.
