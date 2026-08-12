# 06 — Validation and Testing

Three independent layers, because they catch different classes of bug.

| Layer | Runs in | Needs Blender | Catches |
|---|---|---|---|
| Unit tests | uv env | no | detection maths, report logic, contract shape |
| Self-tests | Blender | yes | operator behaviour that cannot be asserted statically |
| Export gate | Blender, every run | yes | structural defects in the written file |

---

## 1. Unit tests

**146 tests, 0.08 s, no Blender.** That is only possible because the detection
maths lives in stdlib-only modules with `bpy` isolated at the edges. Keeping that
seam is what keeps the feedback loop fast enough to actually use.

### What is covered

| Area | Examples |
|---|---|
| Floor detection | peak finding, site plane rejection, parapet roof, mezzanine merging |
| Room detection | flood fill, L-shaped rooms, hairline leaks, slivers, corridors |
| Pole of inaccessibility | including a guard asserting a **centroid would have failed** |
| Plan matching | hashing, IoU, grouping determinism |
| Classification | against the **real measured metrics of every intake object** |
| Ratio solver | pressure curve, class floors, floor-limited reporting |
| Coordinates | Z-up ↔ Y-up round trip |
| Report | status rollup, serialisation, always-writable after a fatal error |

### Two practices worth keeping

**Grids are written as ASCII art.** A room test reads the way the floor plan
looks, so a failure is diagnosable by eye:

```
"..............",
".############.",
".#..........#.",
".############.",
```

**Thresholds are pinned against real measurements.** The classifier is tested
against the actual metrics of every object in the intake asset, so a threshold
change that would wreck the fixture fails in 0.08 s rather than three minutes into
a Blender run.

### Tests that pin a *reason*, not a behaviour

The most valuable test in the suite asserts that a centroid **would have** landed
outside an L-shaped room. It does not test current behaviour — it pins why the
pole of inaccessibility exists, so nobody "simplifies" it back to a centroid.

---

## 2. Self-tests inside Blender

Some things cannot be asserted without rendering.

### Cubemap orientation

The face-orientation table is the highest-risk constant in the pipeline: **a wrong
entry does not raise.** It produces six plausible images that tile into a broken
panorama. No static check can catch it.

The self-test renders a room with six known-coloured emissive walls and asserts
each face shows the right one. Three details keep the test honest, each guarding a
different way it could lie:

1. **A cube, not six placed planes.** A plane needs an orientation, and getting
   *that* wrong would produce failures indistinguishable from the bug being tested
   for. A cube's faces come with correct normals for free.
2. **Wall colours assigned through the same coordinate conversion the renderer
   uses.** A test with its own hardcoded mapping would cancel out a sign error and
   pass.
3. **View transform pinned.** Blender's default desaturates saturated emission far
   enough that cyan reads as white, and the test cannot tell a correct face from a
   wrong one.

The comparison is channel-presence, not exact values — gamma is still in play even
when compression is not.

---

## 3. The export gate

Runs on **every** export. Reads the written file back and asserts on it.

| Check | Catches |
|---|---|
| World bbox vs Blender scene | unit and scale bugs |
| `metallicFactor` explicit on every material | the glTF default-to-1.0 trap |
| Every image resolves | broken texture references |
| Every primitive has `TEXCOORD_0` | meshes that will render untextured |
| Triangle count matches | silent geometry loss |
| File size within budget | product-fit regression |

### The bbox check must walk the node hierarchy

glTF accessor `min`/`max` are in **mesh-local** space; node transforms carry the
unit conversion. Reading accessors alone reports a size that can be tens of times
wrong and sends you hunting a scale bug that does not exist. That happened during
development and cost real time — the check is written to walk nodes for exactly
that reason.

### Not yet wired: the Khronos validator

`glTF-Validator` is the authoritative structural check and should be a hard gate.
It is **not** currently wired, because the npm package has no CLI entry point and
fetching a tool at runtime contradicts the no-network-at-runtime rule.

Correct resolution: **pin the validator binary in the container image** and shell
out to it. Until then the in-process checks cover the failure modes this pipeline
has actually produced, which is not the same as covering the spec.

---

## 4. What is deliberately not automated

| Area | Why | Mitigation |
|---|---|---|
| Visual correctness | No ground truth to diff against | Live Blender session (MCP) for A/B; browser check |
| Panorama seams | Requires human perception | Orientation self-test covers the mechanical part |
| Material appearance | "Looks right" is not machine-checkable | Parity target: match Blender, so differences are bugs |

**A gate that passes is not a promise the model looks right.** It means the file is
structurally sound and dimensionally correct. Visual spot-checking stays necessary
until the pipeline has enough processed models to trust its material and normal
handling specifically.

---

## 5. The gap that matters most

**No fixture with real interiors.** Room detection is built and unit-tested against
synthetic grids, but has never run on a model that actually contains rooms. The
one real asset is an exterior whose only enclosed spaces are two lightwell shafts.

Every tuned number in the room stage — the 4–60 m² range, the 1.2 m minimum
dimension, the 8 m grid spacing — is therefore an estimate validated only against
hand-built grids.

This mirrors `CLAUDE.md` §7's top blocker: **real agency test scenes**. Ideally one
3ds Max + V-Ray and one 3ds Max + Corona, with genuine interior partitions.
