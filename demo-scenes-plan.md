# Demo Scenes — build plan

Three showcase projects built from the assets in `Raw Models/`, authored in Blender
and fed to `blender_core_processing_p0.3` to produce the four-level delivery in
CLAUDE.md §8.

**Purpose is the pitch, not the intake.** These scenes exist to show a prospect what
the product does. Track 1 (3ds Max → FBX → MaxScript) is validated separately, on
real agency files. Nothing here is a pipeline fixture and nothing here should be
used to tune a threshold in §4 — these are clean models and they will lie to you
about how hard the real work is.

**Priority order is: looks good > architecturally correct > fast.**

---

## 0. Blockers — settle these before any modelling

**0.1 Licence.** The three GLBs came with `.zip` siblings, which is the Sketchfab
download shape. Confirm each model's licence before it becomes a public demo:

| Licence | Verdict |
|---|---|
| CC0 | fine, no attribution |
| CC-BY | fine, **visible attribution required in the viewer** |
| CC-BY-NC | not fine — this is commercial sales collateral |
| ND (NoDerivatives) | **blocking** — the whole plan is derivative work |

If any is NC or ND, replace it. Polyhaven (CC0) and the Sketchfab CC0 filter are
both reachable from the Blender MCP, so replacement is cheap. Record the outcome
per model in `demo_scenes/LICENCES.md`. Do this first — it can invalidate a demo
after the work is done.

**0.2 Normalize each model on import.** All three arrive with unapplied transforms;
m3 additionally sits ~50 m off world origin (`min=[26.6, -30.8, -2.7]`). Before
anything else, per model: apply all transforms, recenter to origin on X/Y, sit the
lowest slab on `z=0`, confirm 1 Blender unit = 1 m against a known dimension, weld
(`remove_doubles`, 1e-4). Save as the clean base `.blend`.

**0.3 Strip contamination.** m2 carries materials named `Sumele_Dress`,
`Sumele_Hair`, `Sumele_Jewelry`, `Sumele_Skin` — a human character is bundled into
a building file. Find it and delete it, or drop m2 (see §4).

---

## 1. The interior decision — the biggest change to the stated plan

The plan as described was: *create rooms inside the tower and the complex, then add
filler furniture.* **Don't model interiors into the building meshes.**

§8 defines L3 as **cubemap images**, deliberately decoupled from the building GLB.
The viewer never loads interior geometry. That means the room only has to look
correct **from one eye point**, and it does not have to sit inside the building —
or even be in the same file.

So build **one reusable interior stage set** instead:

- A single 2BHK-ish room shell — living/dining, one bedroom, kitchen edge, balcony
  opening with a view plane.
- Correct ceiling height (3.0 m matches m3's measured slab spacing exactly).
- Dressed with CC0 furniture, lit with an HDRI, one wall of glazing.
- Rendered to cubemaps from 3–4 eye points.

Why this is strictly better here:

- **One interior serves all three demos**, re-dressed for finish level. Modelling
  rooms into two separate towers is the same work three times over.
- **Quality is uncapped.** A standalone room can carry 200K triangles, real glass
  and a full furniture set, because none of it ships as geometry — it ships as six
  compressed images inside a 1.5 MB budget.
- **It cannot break the exterior.** Booleaning partitions into a shell that is
  grouped by material (m3 is 18 objects spanning the whole building) risks wrecking
  the exterior mesh and its UVs.

The building GLBs stay exterior shells. The manifest points L3 hotspots at cubemap
directories; where those images were rendered is invisible to the viewer.

**Honesty line for the demo:** interiors are labelled *representative*, not a
render of that specific unit. That is also true of every builder's show flat.

---

## 2. Shared asset kit — build once, use in all three

Assemble before starting any individual demo. Sources: Polyhaven (CC0, no
attribution) and Sketchfab CC0, both wired to the Blender MCP (§6b).

**Materials** — concrete (smooth + board-formed), painted plaster, brick, glass
(clear + tinted), brushed/anodised metal, timber, stone cladding, paving slab,
asphalt, kerb concrete, lawn, gravel.

**Site kit** — road section with kerb and markings, compound wall segment, gate
(sliding + pedestrian), boundary fence, street light, bollard, signage panel.

**Entourage** — 3–4 tree species, shrub/hedge, planter, 3–4 cars, a few people.
Cheap in triangles, and the single largest jump in perceived quality.

**Interior kit** — sofa, dining set, bed, wardrobe, kitchen run, rug, curtains,
lamps, artwork, houseplants.

**HDRIs** — one clear-morning, one warm-evening, one overcast. Lighting does more
for "looks like an agency render" than any amount of geometry.

Store as a linked-library `.blend` under `demo_scenes/kit/` so all three scenes
reference the same assets rather than duplicating them.

---

## 3. Demo A — "Villa Residency"  *(build first)*

**Base:** `modern_luxury_villa_house_building.glb` — 5,110 tris, 168 objects,
11 textures, 27 × 23 × 12 m, **and it already has a textured interior** (timber
floor and ceiling, brick walls, floating stair).

**Why first:** it is the only asset that already looks finished. It gets you one
complete, beautiful, showable demo in the least time — and one finished demo sells
better than three half-built ones.

**Work:**

1. Fix the 81 of 168 meshes with no UVs — Smart UV Project at `angle_limit=66°`
   per §4 Tier B. Small meshes; low risk.
2. Upgrade materials to full PBR. Currently base-colour only; add roughness,
   normal, and real glass.
3. Build the site: plot subdivision, internal road loop with kerbs, compound wall
   and gate per villa, driveway, lawn, planting, street lights.
4. Instance the villa **10–14 times** with rotation and two or three material
   variants (paint colour, cladding) so it does not read as copy-paste. Total
   ≈ 70K triangles — comfortably inside §4's 80–120K scene budget.
5. Dress two interior rooms with the furniture kit for the L3 capture.

**Levels:** L0 = the whole residency with a low-poly massing version of the villa.
L1 = one villa, per-floor nodes (ground, first). L2 = one floor plate. L3 =
cubemaps in living room and bedroom.

**Watch:** L0 must use a decimated massing villa, not 14 full copies.

---

## 4. Demo B — "Tower Cluster"  *(build last, and consider dropping)*

**Base:** `residential_complex_modern_apartment_building (1).glb` — 328,619 tris,
66 objects, **zero textures**, 13 × 16 × 29 m.

**Be clear-eyed: this is the most work and the least payoff of the three.**

- **Zero images and nothing connected on any material.** Every material is
  `auto_8`…`auto_25`. Full material authoring from scratch.
- **Nearly every object shares one material (`auto_20`).** So walls, glazing,
  balcony slabs and railings cannot be textured differently until the meshes are
  split by face orientation and position. That is the real cost, and it is not
  small.
- **328K triangles for the smallest building of the three.** Replicating five
  towers is 1.6M triangles — an order of magnitude over §8's budget. It must be
  decimated hard (target ≤ 60K per tower) *before* replication.
- Floor histogram is irregular (0, 4.0, 5.5, 7.5, 9.5, 11.5, 13.5, 15.0, 17.0 …) —
  roughly 2 m spacing, which is below storey height, so balcony slabs are being
  read as floors. Floor detection will need help here, unlike m3.
- Carries the stray character materials (§0.3).

**If built:** decimate → split by orientation → author materials → replicate 4–5
towers around a central landscaped garden with paths, water feature, seating,
planting. The cluster-with-garden idea is good and it is the one thing this model
offers that m3 does not.

**Recommendation:** build A and C to a finish first. If a third demo is still
wanted, consider **re-using m3** at a different massing and finish instead — it is
cleaner in every measurable way. Decide after A and C are done, not now.

---

## 5. Demo C — "Mixed-use Complex"  *(build second — the commercially important one)*

**Base:** `residential_complex_modern_apartment_building.glb` — 192,960 tris,
18 objects, 8 textures, **49 × 88 × 45 m**, 17 of 18 meshes already UV'd.

**Why it matters most:** this is what Pune developers actually build and what your
target agencies actually render. It is the demo that makes a prospect see their own
work.

**Its structural gift** — the floor histogram is textbook:

```
z = 3.5  6.5  9.5  12.5  15.5  18.5  21.5  24.5  27.5  30.5  33.5  36.5
```

Twelve storeys at **exactly 3.0 m**, with identical slab area at every level.
P0.3's floor detection will land this cleanly and the plan-dedup path (identical
storeys captured once) will fire.

**Work:**

1. **Retexture with Polyhaven PBR.** Only 7 of 15 materials have even a base
   colour; there is no roughness, metallic or normal map anywhere. This is the
   single biggest visual win available in the whole plan.
   - Bounded, because the model is grouped *by material* into 18 objects and 17
     already have UVs.
   - **The fiddly part is texel density.** Existing UVs were authored for different
     maps, so a tiling brick may land 3 m wide. Check and correct scale per
     material against a known dimension before judging any material.
2. Real glass on the glazing (currently flat blue) — the highest-impact single fix.
3. Ground plane: podium paving, landscaping, kerb, cars, trees. A building floating
   on grey reads as a student model.
4. **Slice L1 into per-floor nodes** by bisecting at the twelve 3.0 m slab planes.
   §8 requires per-floor named nodes at L1 and per-unit meshes at L2, and the model
   as delivered is grouped by material spanning the full height — no object
   corresponds to a floor. This is the one genuine geometry task in Demo C.
5. L3 from the shared interior kit (§1) — no rooms modelled into the shell.

**Note:** object names are 3ds Max defaults (`Cone001`, `Box002`, `Line003`), so
§4's name regex will match 0%, exactly as measured in P0.2. Geometry heuristics
carry classification. Expected, not a bug.

---

## 6. The quality bar — what "looks good" actually means here

Stated priority is appearance, so these are requirements, not polish:

- **HDRI lighting, never a bare sun lamp.** Largest single quality delta.
- **Real glass** — reflection and roughness. Flat blue windows are the clearest
  tell of an amateur render.
- **Bake AO into the ORM red channel.** §4 already mandates ORM packing
  (AO→R, Roughness→G, Metallic→B) and that R channel is currently unused. These are
  static scenes, so baked AO and contact shadows cost nothing at runtime and buy a
  large amount of perceived depth. Highest quality-per-byte item on this list.
- **Ground and entourage always.** Paving, planting, a few cars and trees.
- **Consistent texel density** across a scene — one wrongly-scaled brick undoes
  everything else.
- **Silhouette survives decimation.** Check thin features (railings, mullions,
  parapets) after decimation; use planar dissolve, not collapse, per §4.

---

## 7. Output contract — per demo

Each demo produces the §8 four-level set through P0.3, which already accepts
`.blend` input (`p03/cli.py`), so authored scenes feed the existing pipeline
unchanged:

| Level | Artefact | Budget |
|---|---|---|
| L0 | masterplan GLB, low-poly massing, merged | ≤ 8 MB |
| L1 | one building, **per-floor named nodes** | ≤ 5 MB |
| L2 | one floor plate, **per-unit meshes** | ≤ 2 MB |
| L3 | cubemap image set per room | ≤ 1.5 MB |

Plus `manifest.json` with a **world-space bbox per node**, `report.json`, and
**stable IDs**. Current hotspot IDs (`f0_h0`) are positional and can silently
repoint after reprocessing — §8 flags this as blocking the first public URL, and
these demos will be the first public URLs. Fix the ID scheme before publishing,
not after.

Also required before a prospect sees it: KTX2 (hard-blocking per §8 — the budgets
above are unreachable with raw JPEG), and vendored self-hosted Draco/KTX2 decoders
(a missing decoder presents as an empty scene with **no console error**).

---

## 8. Order and rationale

1. **§0 blockers** — licence and normalization. Can invalidate everything after.
2. **§2 shared kit** — build once; all three demos draw from it.
3. **§1 interior stage set** — one room, dressed and lit, cubemaps out.
4. **Demo A (Villa Residency)** — fastest route to one finished, good-looking demo.
5. **Demo C (Mixed-use Complex)** — the commercially important one.
6. **Demo B (Tower Cluster)** — only if still wanted after A and C ship.

A finished Demo A in hand beats three scenes at 60%. It is also what gets carried
into a prospect meeting, and the meeting is the point.

---

## 9. Risks

| Risk | Mitigation |
|---|---|
| Model licence forbids derivative/commercial use | §0.1, before any work |
| m2 material splitting balloons in scope | Timebox; drop m2 per §4 |
| Replication blows the L0 triangle budget | Separate low-poly massing per demo |
| Texel density wrong after retexture | Check against a known dimension first |
| Slicing m3 into floors damages exterior UVs | Bisect at slab planes; verify UVs after |
| Demo tuning leaks into §4 thresholds | These are clean models — never tune §4 here |
