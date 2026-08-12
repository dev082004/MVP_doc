# ADR-0004 — UV transfer is recovery-only, never an improvement pass

**Status:** Accepted
**Date:** 2026-08-08

## Context

The documented optimization strategy prescribes preserving UVs through decimation
by snapshotting the full-resolution mesh, decimating, then projecting the original
UVs back with a Data Transfer modifier. This is the standard technique and it is
what the research specifies.

Applied unconditionally, **it destroyed the UVs it was meant to protect.**

Measured, same decimation, with and without the transfer pass — total UV area
change:

| Object | Decimate alone | Decimate + Data Transfer |
|---|---|---|
| `Line003` | −16.0% | **+21,311.5%** |
| `Cone001` | +0.0% | +222.0% |
| `Box414` | −45.7% | +173.0% |

### Why

The transfer used nearest-polygon interpolation, which is incoherent once UVs
tile. This asset tiles heavily — `Line003` spans u ∈ [−13.39, 14.39], roughly 27
repeats. Two adjacent destination loops can sample source faces on *different
tiles*, one at u ≈ 2.1 and one at u ≈ 3.9, and interpolating between them smears
the texture across the whole range.

Topology-based mapping is not an option either: it requires the two meshes to
correspond face-for-face, which is exactly what decimation has just destroyed.

### The counter-measurement

Blender's Decimate modifier **already interpolates UVs**, and it does the job. On
`Cone001` — 115,872 triangles — collapse changed total UV area by **0.0%**.

The transfer pass was solving a problem that did not exist, and creating a much
larger one.

## Decision

Run the transfer **only when decimation actually dropped the UV layer**
(`len(uv_layers) == 0` afterwards). Never to "improve" UVs that survived.

Record the outcome per object as a tier:

| Tier | Meaning |
|---|---|
| `A` | Had UVs, survived decimation. The common case. |
| `A-restored` | Had UVs, decimation dropped them, transferred back from the donor. |
| `A-failed` | Had UVs, lost them, unrecoverable. An error. |
| `B` | Never had UVs; unwrapped procedurally. |
| `B-failed` | Never had UVs and unwrapping failed. |

Guard the whole stage with a **measurement, not trust**: compute total UV area
before and after, and raise `UV_DISTORTED` if it moves outside a sane band.

| Bound | Value | Why |
|---|---|---|
| Minimum | 0.35× | Decimation legitimately shrinks UV area a little as extreme corners are removed |
| Maximum | 1.15× | Nothing legitimate *grows* a UV layout |

## Consequences

- The full-resolution donor mesh is still created when UVs exist, purely as
  insurance. It costs a mesh copy and is usually unused.
- The UV area check is the only thing standing between a scrambled mapping and a
  shipped model. **Triangle counts stay perfectly plausible while the mapping
  underneath is destroyed**, so nothing else notices.
- With decimation now off by default ([ADR-0005](0005-decimation-opt-in.md)), this
  entire hazard is absent from the default path.

## Alternatives considered

**Topology mapping, as documented.** Rejected — it needs face-for-face
correspondence that decimation has just eliminated.

**Detect tiling and only transfer on non-tiled objects.** Plausible, and would
recover the technique for some meshes. Rejected as unnecessary: measurement shows
Decimate's own interpolation is adequate, so the transfer buys nothing on the
happy path.

**Drop the donor entirely.** Rejected — decimation *does* occasionally drop the UV
layer outright, and without a donor that object is unrecoverable.

## Notes

The general lesson: **the prescribed technique was correct in the abstract and
wrong for this data.** It only surfaced because UV area was measured rather than
assumed. A stage that "looks like it worked" and has a plausible triangle count
can still be catastrophically broken — the check has to target the property that
actually matters.
