# ADR-0006 — Validate the written GLB, not the scene

**Status:** Accepted
**Date:** 2026-08-08

## Context

Two defects reached a delivered GLB. Neither raised an exception, both left
perfect triangle counts, and no test caught either.

**1. Every material fully metallic.** `metallicFactor` was omitted from 14 of 15
materials. The glTF default is `1.0`, so every surface became a dark mirror. The
export was a *correct encoding of corrupt input* — nothing in the writing was
wrong.

**2. Meshes with no UVs.** Making a stage optional removed procedural unwrapping
from the default path, and two meshes exported with no `TEXCOORD_0`. They render
untextured regardless of material quality. Triangle count: unchanged and correct.

Both are **structural properties of the written file**. Asserting on the Blender
scene would have caught neither, because in both cases the scene faithfully
matched what was written.

There was also a near-miss in the other direction: a bbox check that read glTF
accessor `min`/`max` without applying node transforms reported the model as ~75×
oversized. It sent a real investigation after a scale bug that did not exist.
Accessor bounds are **mesh-local**; the node TRS carries the unit conversion — on
this asset a `0.0254` scale, because the source FBX is in inches.

## Decision

After every export, **read the written file back and assert on it.**

| Check | Catches |
|---|---|
| World bbox vs Blender scene, within tolerance | unit and scale bugs |
| `metallicFactor` present on every material | the glTF default-to-1.0 trap |
| Every image has embedded data or a URI | broken texture references |
| Every primitive has `TEXCOORD_0` | meshes that will render untextured |
| Triangle count matches the scene | silent geometry loss |
| File size within an advisory ceiling | product-fit regression |

Failures are diagnostics, not exceptions — the model still ships as `partial`,
consistent with the error model.

**The bbox check walks the node hierarchy and applies every transform.** This is
not optional and is commented as such in the code, because doing it the obvious
way produced a convincing false alarm.

## Consequences

- The gate parses the GLB with stdlib only. Cheap, no dependency, no network.
- It validates the plain export. Draco rewrites buffers but not the JSON structure
  the gate inspects, so one pass covers both variants.
- **A gate that passes is not a promise the model looks right.** It means the file
  is structurally sound and dimensionally correct. Visual checking stays necessary.
- Every new class of shipped defect should become a new check. The gate is meant to
  grow.

### Not yet wired: the Khronos glTF-Validator

The official validator is the authoritative structural check and should be a hard
gate on top of these checks. It is not wired because the npm package exposes no
CLI entry point, and fetching a tool at runtime contradicts the
no-network-at-runtime rule.

Correct resolution: **pin the validator binary in the container image.** Until
then these checks cover the failure modes this pipeline has actually produced,
which is explicitly not the same as covering the specification.

## Alternatives considered

**Assert on the Blender scene before export.** Rejected — it would have caught
neither shipped defect. Both scenes were correct; the encoding was the problem.

**Rely on the viewer to surface problems.** Rejected — that is manual visual QA as
the primary catch, it does not scale, and it puts the discovery after delivery.

**Only validate in tests.** Rejected — the defects came from real agency input, and
tests run on fixtures. The gate has to run on every job.

## Notes

The guidance this follows says it plainly: *a model that passes this gate but is
still visually broken is a spec gap — tighten the checks rather than relying on
manual visual QA as the primary catch.* Both defects above became checks.
