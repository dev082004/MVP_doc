# ADR-0005 — Decimation is opt-in; planar dissolve is repair

**Status:** Accepted
**Date:** 2026-08-08

## Context

Triangle reduction was the pipeline's headline feature and ran on every job. On
the first real asset it cost more than it returned.

With a 30%-keep target requested:

| | |
|---|---|
| Achieved | 41% kept (191,310 → 78,348) |
| Outcome | floor-limited; target unreachable |
| Cost | one `UV_DISTORTED` error, thin balusters shattered |
| Reason | **83% of the scene is thin geometry** that breaks under collapse |

So the stage failed to hit its target *and* damaged the model on the way. Fidelity
was the immediate priority — the delivered model did not yet look like Blender —
and every UV hazard in the optimizer exists because decimation moves geometry.

### The distinction that resolves it

Two operations were bundled under "decimation" and they are not the same kind of
thing:

| Operation | Moves vertices? | Nature |
|---|---|---|
| **Collapse** | yes | Lossy reduction — trades shape for triangles |
| **Planar dissolve** | **no** | Lossless repair — merges coplanar faces |

Planar dissolve removes *redundancy*, not detail. Measured:

| | Triangles |
|---|---|
| After weld | 191,310 |
| After planar dissolve (5°) | **107,956** |
| `Cone001` alone | 115,872 → 37,536 |

**44% of the scene's triangles were redundant coplanar subdivision** — geometry
describing no shape. Removing it is repair by any reasonable definition.

## Decision

**Collapse decimation is opt-in** (`--decimate`), off by default.

**Planar dissolve runs on the default path** as a repair step, alongside welding.

Keep the full per-class solver, ratios, floors and tolerance logic intact — it is
correct work and it will be needed. It simply does not run unless asked.

## Consequences

- Default output is substantially larger in triangles. On the test asset: 191,310
  rather than 78,348. That is over the 80–120K mobile budget and is a **known,
  deliberate regression** while material and texture fidelity is the priority.
- Planar dissolve should bring the default path to ~108K without lossy reduction —
  inside the budget on this asset, though that will not generalise.
- `UV_DISTORTED` and the whole nearest-polygon transfer hazard
  ([ADR-0004](0004-uv-transfer-recovery-only.md)) disappear from the default path.
- Textures, not triangles, are where the file size actually is: **82% of the
  exported bytes**. Reduction was never going to fix the size problem.

### One regression this caused, and its lesson

Turning decimation off silently shipped two meshes with **no UVs at all**, because
procedural unwrapping had been implemented inside the optimizer. A mesh without
UVs cannot be textured no matter how good its materials are.

Unwrapping moved to intake, where it belongs — it is a fidelity requirement, true
whether or not anything later reduces the triangle count. The export gate gained a
`MESH_WITHOUT_UV` check so the same class of bug cannot ship again.

**Making a stage optional exposes every responsibility that was hiding inside it.**
Audit what else lives there before flipping the switch.

## Alternatives considered

**Keep decimation on, at a gentler ratio.** Rejected — it still moves geometry and
still risks UVs, for a saving that textures dwarf.

**Remove the decimation code entirely.** Rejected — the per-class solver is sound
work, is unit-tested, and will be needed once fidelity is settled. Deleting it
would mean rebuilding it.

**Treat planar dissolve as decimation and gate it too.** Rejected — it is
lossless. Gating it would discard 44% of the triangles for free and improve
shading, for no reason.

## Notes

Reversing this decision is expected. Once material and texture parity is settled
and there is a real interior fixture, decimation should come back on by default —
with the class floors informed by however the thin-feature handling has evolved.
