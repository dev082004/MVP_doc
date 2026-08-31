# Improvements & Observations — every problem found, and the ones coming

A complete list of what has gone wrong across P0.1 → P0.4, why it went wrong, and
what to do about it. Plus the problems that have **not** happened yet but will,
once real client files arrive.

Written in plain language. Where a technical word is needed it is explained the
first time.

---

## How to read this

Every problem is one block:

> **[ID] Short name**
> **What happens** — the symptom you would actually see.
> **Why** — the real cause underneath.
> **Found in** — which version, or "predicted" if we have not hit it yet.
> **Fix** — realistic and general, not a one-off patch.
> **How to catch it automatically** — because finding it by eye does not scale.

**Severity:**

| | meaning |
|---|---|
| 🔴 **BLOCKER** | ships a broken or wrong-looking result to a client |
| 🟠 **MAJOR** | wastes real money or time, or hides other bugs |
| 🟡 **MINOR** | annoying, cheap to fix, worth doing |
| 🔵 **CONCEPT** | not a bug — a design decision that is wrong or missing |

**Status:**

| | meaning |
|---|---|
| ✅ | fixed |
| ⚠️ | partly fixed, or fixed in one version only |
| ❌ | open |
| 🔮 | predicted, never encountered |

---

## The single most important observation

Across four versions, the pattern that repeats is not "the code had a bug". It is:

> **A stage silently did nothing, or did something wrong, and nothing noticed
> until several stages later.**

- Decimation removed nothing, because the mesh was not welded. *(P0.2)*
- UV transfer destroyed UVs while reporting success. *(P0.2)*
- The colour-space fix corrupted the building. *(P0.2)*
- Lighting was skipped because one lamp existed. *(P0.3)*
- The AO texture was never packaged. *(P0.4)*
- An object was decimated out of existence. *(P0.4)*
- Blender hung for 12 minutes using no CPU. *(P0.4)*

**Not one of these raised an error.** Every one presented as "the output looks a
bit wrong" much further downstream.

So the highest-value structural improvement in the whole project is not any
individual fix below. It is this:

> **Every stage must assert that it actually did what it claimed, and every
> number in the report must be measured after the fact rather than predicted.**

If decimation says "reduced 60%", it should measure the mesh afterwards and
compare. If a texture is packaged, the package should be re-opened and checked.
Cheap to add, and it converts an entire class of silent failures into loud ones.

---

# PART A — Intake: what arrives from the client

This is where the real work is. The pipeline's difficulty is not clever geometry
maths — it is that **real files are a mess** and every one is messy differently.

Our test model came from a stock library and is comparatively tidy. A real
agency file will be far worse. Sections marked 🔮 are anticipated, and the
reasoning for each is given.

### A1 🔴 ✅ Mesh arrives as loose disconnected triangles

**What happens** — Decimation runs, reports success, removes almost nothing. The
file stays enormous. Every rule based on counting "pieces" of a mesh gives
nonsense.

**Why** — 3ds Max's FBX exporter writes each triangle with its own three
vertices, sharing nothing. One object arrived as 115,872 triangles across
115,872 separate pieces — 347,616 vertices for a real 58,752. Collapse
decimation works by merging shared edges. If nothing is shared, there is nothing
to merge, so it does nothing at all — **without complaining**.

**Found in** — P0.2.

**Fix** — Weld (merge vertices that sit in the same place) as the *first*
operation after import, always, before anything else looks at the mesh. Costs
about 1% of triangles.

**How to catch it** — After import, compare the number of pieces to the number
of triangles. If they are roughly equal, the mesh is exploded. After welding,
assert the vertex count actually dropped when it should have.

**Note** — P0.4 could not test this, because its file came from Blender, which
writes clean meshes. The code is right but has never met the problem. **A real
3ds Max file is the top testing priority.**

---

### A2 🔴 ✅ Two FBX importers, two different answers

**What happens** — The model looks fine in Blender and nearly black in a
browser.

**Why** — Blender ships two FBX importers. On the same file, the old one
reported every material as **100% metallic**; the new one reported 0%. A fully
metallic surface has no base colour at all — it shows only reflections. Blender's
viewport has a built-in studio environment so it looked like plausible brushed
metal; a browser with plainer lighting rendered it almost black.

Worse, the export was *correct*: glTF treats a missing metallic value as 1.0, so
we faithfully wrote down corrupt input.

**Found in** — P0.2.

**Fix** — Pin one importer explicitly and never rely on the default. Record which
one was used in the report.

**How to catch it** — After import, check the spread of material values. If
*every* material has an identical, extreme value (all metallic 1.0, all roughness
0.859), that is not a coincidence — that is an importer inventing data. Flag it.

---

### A3 🔴 🔮 Missing texture files

**What happens** — Materials come out flat grey. Roughly a third of real
submissions are expected to have at least one broken image link.

**Why** — The artist's file references `D:\Work\Client\textures\brick.jpg`. That
drive does not exist on our server, and often does not exist on their new laptop
either. Files get moved, renamed, or left on a colleague's machine.

**Found in** — Predicted. P0.1 already has a `MISSING_TEXTURE` diagnostic, and
P0.4's audit blocks on it, but no real broken file has been processed.

**Fix** — Three layers:
1. The agency-side script must **copy every texture into the package**, not
   reference it. Already done.
2. Before packaging, verify every referenced file exists and is readable.
3. If some are missing, **still ship the package** with those materials marked
   failed and substituted with a sensible grey. Never refuse the whole job over
   three of fifty materials.

**How to catch it** — Automatic, at export. Report the count and the exact
material names so the artist can fix them in one pass rather than one at a time.

---

### A4 🟠 🔮 Textures in formats we cannot read

**What happens** — Texture is present, and still comes out grey.

**Why** — Arch-viz artists routinely use **PSD (Photoshop) files directly as
textures** in 3ds Max. Also common: TGA, TIFF with unusual compression, 16-bit
and 32-bit images, EXR, and occasionally a `.max` internal bitmap. Blender cannot
read PSD at all. Web browsers can read almost none of these.

**Found in** — Predicted, and high confidence. This is normal practice in the
industry.

**Fix** — At the agency side, **convert every texture to PNG or JPEG during
export**, using the host software (3ds Max *can* read its own PSDs and re-save
them). Record the original format. Anything unconvertible is reported as a failed
material rather than silently dropped.

**How to catch it** — Check file extensions and actually try to open each image
during export. Do not trust the extension alone — a `.jpg` that is really a PSD
is a real occurrence.

---

### A5 🟠 🔮 Non-English and duplicate names

**What happens** — Files fail to write, or two different materials collapse into
one, or a path breaks on a different operating system.

**Why** — Indian studios name things in Hindi and Gujarati, use spaces and
slashes, and reuse names constantly. "Material #1" might exist four times with
four different appearances.

**Found in** — Predicted, but already designed for.

**Fix** — Already handled by two decisions:
- Textures are named after their content checksum, so the original filename never
  touches the filesystem.
- Every object and material gets a generated unique ID that travels alongside the
  human name.

Filenames derived from material names are sanitised to safe characters.

**How to catch it** — Check for duplicate names at audit and report them, because
the *artist* needs to know even though we can cope.

---

### A6 🔴 🔮 Wrong units

**What happens** — The building is 30 millimetres tall, or 30 kilometres tall.
Every threshold in the pipeline — storey height, room size, weld distance — is
suddenly meaningless.

**Why** — 3ds Max scenes are commonly authored in centimetres, millimetres,
inches, or "generic units" with no meaning at all. Expected in perhaps 40% of
submissions.

**Found in** — Predicted. P0.4's audit checks that units are *set*, but nothing
verifies they are *sensible*.

**Fix** — Two-part:
1. Record the declared unit scale at export and normalise to metres.
2. **Sanity-check against reality.** A residential building is 2.4–4.5 m per
   storey and 3–200 m tall. If the model says the total height is 0.04 or 40,000,
   the declared unit is wrong regardless of what the file claims. Guess the most
   likely correct scale, apply it, and flag loudly.

**How to catch it** — Compare the model's overall size against known-plausible
building dimensions. This is one of the highest-value automatic checks available
and it is not yet implemented.

---

### A7 🟠 ✅ An object absurdly larger than the building

**What happens** — Two separate downstream failures: depth precision breaks in
the browser (surfaces flicker against each other), and any automatic camera
framing puts the building at about 40 pixels tall.

**Why** — Our test model contained a backdrop ground plane **108 km across** made
of two triangles. Two triangles cost nothing, so no weight check saw it. Real
scenes routinely contain a huge sky dome, ground plane, or a stray object
accidentally scaled by 1000.

**Found in** — P0.4.

**Fix** — Detect objects far larger than the median object size, tag them as
"context", and clamp them to a sensible multiple of the building footprint.

**How to catch it** — Compare each object against the **median** object size.
Not the average and not the overall scene size — both get dragged upward by the
very object you are hunting, so **the outlier helps hide itself**.

---

### A8 🟠 ✅ Distance from origin ≠ size of scene

**What happens** — An audit designed to catch "scene is in the wrong place"
passed a scene that was catastrophically wrong.

**Why** — The 108 km plane was **centred**, so the scene's middle sat innocently
19.8 m from zero while its edges were ±54 km. The rule measured the wrong thing.

**Found in** — P0.4.

**Fix** — Offset and extent are **different failures** and need separate checks.
Both are now implemented.

---

### A9 🔴 🔮 Model built far from the origin

**What happens** — Surfaces shimmer and jitter as the camera moves, especially
on a phone.

**Why** — Models imported from surveying data or CAD are often placed at real
world coordinates — 500,000 metres from zero. Graphics hardware stores positions
with limited precision, and that precision is *relative to zero*. Far from zero,
the gaps between representable positions become larger than the details you are
trying to draw.

**Found in** — Predicted. P0.4 has a rule but no real file has triggered it.

**Fix** — Detect the offset at import, move everything back to near zero, and
**record the offset** in the manifest so any real-world positioning can be
restored later.

**How to catch it** — Automatic and cheap. Already written.

---

### A10 🟠 🔮 Geometry that lives in another file

**What happens** — Objects arrive empty. Trees, cars, furniture — all missing.

**Why** — V-Ray Proxy and XRef objects are placeholders that load their real
geometry from a separate file at render time. FBX does not carry them. Arch-viz
scenes are full of them, because that is how you fit 400 trees into a scene.

**Found in** — Predicted. P0.4's audit blocks on it.

**Fix** — The agency-side script must **resolve proxies to real mesh at preview
resolution** before exporting. 3ds Max can do this. Failing that, export just
their positions and substitute a simple stand-in server-side — for vegetation, a
flat billboard is genuinely the right answer anyway.

**How to catch it** — Scan for proxy object types at audit. Also flag objects
with zero triangles but a non-zero bounding box, which is what an unresolved
proxy looks like after import.

---

### A11 🟡 🔮 Hidden geometry, and geometry nobody meant to send

**What happens** — The file is twice the size it should be, and the triangle
budget is spent on invisible things.

**Why** — Real working files contain: earlier versions of the building parked off
to one side, the client's old logo, a colleague's test object, hidden layers,
construction guides, and lighting rigs. Artists hide things rather than deleting
them.

**Found in** — Predicted, though P0.4 did hit the related case of boolean cutter
objects (see A12).

**Fix** — Do not export hidden objects, but **report what was skipped** — the
artist may have hidden something by accident. Also flag any object sitting far
away from the main cluster of geometry, which is the signature of "parked to one
side".

---

### A12 🔴 ✅ Construction helper objects re-fill the holes they cut

**What happens** — Every window in the building is filled in with a solid block.

**Why** — Windows are made by subtracting boxes from the facade. Those boxes stay
in the scene. Exporting them puts them right back where the holes are.

**Found in** — P0.3, and handled in P0.4.

**Fix** — Detect helper objects (naming conventions like a `__` prefix, or words
like cutter/boolean/dummy) and exclude them at export. Crucially, exclude by
*filtering the export selection*, never by deleting from the artist's scene.

**How to catch it** — Naming works here, unlike for regular geometry (see D1),
because a `__` prefix is a deliberate authoring convention rather than a default
name. But it is not reliable across studios, so it should be backed by a
geometric check: an object that sits exactly inside the holes of another object
is very likely a cutter.

---

### A13 🟡 🔮 FBX version and flavour differences

**What happens** — Import fails, or produces subtly different results between two
client files.

**Why** — FBX has many versions (2011 through 2020+) and both binary and text
forms. Text FBX files are roughly ten times bigger and much slower to read. Older
versions store materials differently.

**Fix** — Accept whatever arrives, record the version in the report, and pin the
importer. If a file is text FBX, note it — a 2 GB text FBX is a real possibility
and will need a longer timeout.

---

### A14 🟠 🔮 One file containing several buildings

**What happens** — Floor detection finds twelve storeys that do not line up,
because it is looking at three towers of different heights at once.

**Why** — Agencies model whole townships in one file. Our entire design assumes
"one building, one file" for Phase 1.

**Fix** — Detect separated clusters of geometry at intake and either (a) process
the largest and report the rest as ignored, or (b) split into separate jobs. Do
not attempt to process all of them as one building — every height-based rule will
break.

**How to catch it** — Cluster objects by horizontal position. Several
well-separated clusters of similar height means several buildings.

---

# PART B — Geometry and transformation

### B1 🔴 ✅ Surfaces flicker against each other

**What happens** — Balcony slabs and facade panels shimmer and swap in and out as
the camera moves.

**Why** — Two causes together. First, arch-viz models are full of surfaces that
sit in exactly the same place (a slab drawn twice, or a panel flush against a
wall). Second, the browser's depth buffer had a near/far range of 22,954:1, which
gives only about **6 mm of depth precision at 90 m distance**. Two surfaces 2 mm
apart are indistinguishable, so the renderer picks arbitrarily and the choice
changes as you move.

**Found in** — P0.2.

**Fix** — Two parts, both needed:
- Viewer side: use a logarithmic depth buffer and push the near plane out. Never
  set the near plane to a tiny value "just in case".
- Pipeline side: detect coincident faces and nudge or remove one. **Not yet
  implemented.**

**How to catch it** — Look for faces that overlap in space within a millimetre.
This is a known-hard computation but a cheap approximation (compare face centres
and normals) catches most real cases.

---

### B2 🟠 🔮 Flipped normals — surfaces invisible from the outside

**What happens** — Parts of the building are invisible, or you can see through
the outside wall into the interior.

**Why** — Every face has a front and a back. Most viewers do not draw backs, to
save time. CAD imports and hand-modelled geometry frequently have faces pointing
the wrong way. In interiors this is genuinely ambiguous — "outward" has no clear
meaning inside a closed room.

**Found in** — Predicted. Not implemented at all.

**Fix** — Recalculate normals so they point outward, using the object's own
volume as the reference rather than the scene's. For SketchUp files, use its
front/back material assignment as the hint, since SketchUp tracks this
explicitly. As a fallback for genuinely ambiguous cases, mark the material
double-sided — costs performance but is never invisible.

**How to catch it** — Cast rays from outside the object; if most hit back-faces,
the object is inside-out.

---

### B3 🟠 ✅ Negative and uneven scaling

**What happens** — An object renders inside-out, or textures appear mirrored.

**Why** — Artists mirror objects by scaling them by −1. This flips which side of
each face is the front. Uneven scaling (2, 1, 1) also distorts normal maps.

**Found in** — P0.4's audit flags it (one object in the test model). **Nothing
fixes it yet.**

**Fix** — At export, "reset transform" — bake the scaling into the mesh itself so
the object's own transform becomes neutral. This is a standard, reversible 3ds
Max operation. Then recalculate normals.

---

### B4 🟡 🔮 Junk geometry

**What happens** — Small artifacts, weird shading, and decimation behaving
unpredictably.

**Why** — Real meshes contain zero-area faces, zero-length edges, vertices
attached to nothing, and extremely long thin "sliver" triangles. CAD conversions
produce these by the thousand.

**Found in** — P0.1 has `DEGENERATE_FACES` and `LOOSE_VERTICES` diagnostics, but
they only report.

**Fix** — A proper cleanup pass before welding: dissolve degenerate faces, delete
loose vertices and edges. The tolerance must be **derived from the scene's size**,
not hardcoded — a tolerance appropriate for a building would destroy a doorknob.

---

### B5 🟠 ⚠️ Sliver triangles plus smooth shading look terrible

**What happens** — Flat walls show strange dark streaks and banding.

**Why** — Very long thin triangles have unreliable direction information. Applying
smooth shading across everything blends those unreliable directions into
neighbouring good ones and smears the error across the surface.

**Found in** — P0.2 (recorded as a high-priority open limitation).

**Fix** — Do not apply smooth shading universally. Use an angle threshold so
genuinely flat surfaces stay flat and only real curves get smoothed. Also, split
slivers or remove them during cleanup.

---

### B6 🔴 🔮 Walls with no thickness

**What happens** — Walls disappear when viewed from one side; rooms cannot be
detected; interiors leak into the exterior.

**Why** — Many models draw a wall as a single flat sheet with no thickness,
because from the outside it looks the same and it is half the geometry. Our room
detection works by slicing the building horizontally and flood-filling the empty
space — a zero-thickness wall may not reliably block that fill.

**Found in** — Predicted. Room detection already dilates (thickens) walls before
filling, which mitigates this, but it has never been tested against a real
zero-thickness model.

**Fix** — The dilation step already helps. Beyond that, mark such walls
double-sided so they are at least visible from both directions.

---

### B7 🟡 🔮 Duplicated geometry sitting in the same place

**What happens** — The file is bigger than it should be and surfaces flicker
(see B1).

**Why** — Copy-paste accidents. An object pasted twice at the same position looks
identical to one object, and nobody notices until the triangle count is double
what it should be.

**Fix** — Compare objects by a cheap fingerprint (triangle count, bounding box,
volume). Identical fingerprints at the same position mean a duplicate. Report it,
remove the copy.

---

### B8 🟡 ❌ Interior faces nobody will ever see

**What happens** — Triangle budget spent on the insides of solid objects.

**Why** — A "solid" wall modelled as a box has an inside surface that is never
visible. Multiply by every object in the scene.

**Fix** — Cast rays from outside the building and mark faces that are never hit.
**High risk** for interior walkthroughs, where "never visible from outside" is
exactly wrong. Enable only for exterior-only deliverables.

---

# PART C — Materials and textures

### C1 🔴 ✅ Deciding what a texture *is* from its filename corrupts the model

**What happens** — The building's colours go badly wrong after a change that was
supposed to fix colour handling.

**Why** — Data images (roughness, metallic, normal) must be read as raw numbers;
photographs must be read with a brightness curve applied. The obvious way to tell
them apart is the filename. It fails badly: in the real test model, **five of
seven textures were named `_AmbientOcclusion`, `_Displacement`, `_METALNESS` — and
every one of them was plugged into the colour slot.** The artist dragged whatever
map looked good into the diffuse channel.

> The filename says what the image *was*. Only the wiring says what it *is being
> used as*.

**Found in** — P0.2.

**Fix** — Decide from how the image is connected, not from its name. Use the name
only as a last resort when nothing conclusive is wired.

**Related trap** — When matching names, anchor the pattern at word boundaries.
Unanchored, "ao" matches half of English and "orm" matches "platform", "normal",
and "transform".

---

### C2 🔴 ✅ The recommended UV-preservation technique destroys UVs

**What happens** — Textures come out smeared and scrambled after decimation.

**Why** — Standard advice is: copy the mesh, decimate the copy, then project the
original texture coordinates back on. Measured result on the real model:

| object | UV area change without the "fix" | with it |
|---|---|---|
| `Line003` | −16.0% | **+21,311.5%** |
| `Cone001` | +0.0% | **+222.0%** |
| `Box414` | −45.7% | **+173.0%** |

The technique matches each new point to the nearest old face. That breaks down
completely once textures tile — and this model tiles 27 to 55 times over. Two
neighbouring points end up sampling different repeats of the texture and the
mapping explodes.

The counter-measurement is the important part: **Blender's decimation already
handles UVs adequately on its own** — 0.0% change on a 115,872-triangle object.
The fix was solving a problem that did not exist and creating a large one.

**Found in** — P0.2.

**Fix** — Do not run it by default. Keep it as a recovery option only.

**General lesson** — **Measure whether a repair actually improved anything.** If
it did not, remove it. Several "best practices" in the research documents are of
this kind.

---

### C3 🔴 ✅ Bump maps vanish silently on export

**What happens** — Surface detail disappears between the source and the web file,
with no error.

**Why** — 3ds Max and Blender both support "bump" inputs, which fake detail from
a grey height image. glTF has no equivalent. Blender's exporter **throws them
away without a message**.

**Found in** — P0.4.

**Fix** — Detect bump inputs during material reading and mark those materials as
needing conversion. Convert the height image into a proper normal map — a
straightforward calculation that measures the slope of the grey image.
**Detection is implemented; the conversion itself is not.**

---

### C4 🔴 ⚠️ Procedural materials cannot travel

**What happens** — A surface that looked like detailed concrete arrives as flat
grey.

**Why** — Some materials are generated by maths at render time rather than stored
as an image. glTF has no way to carry that. Real V-Ray and Corona scenes use
these for concrete, tiles, noise, scratches, and very commonly **Falloff maps**
for glass and fresnel effects.

**Found in** — P0.4 detects and correctly classifies these as "must bake", but
**baking is not implemented**, so they export flat. This is currently visible in
our own test output as a featureless ground plane.

**Fix** — Implement baking: render the material out to an image file at a
sensible resolution, then treat it as an ordinary image. Expensive (10–30× the
processing cost), which is exactly why it must be reserved for the small number
of materials that genuinely need it.

**Priority note** — This is currently the **largest known visual gap** in P0.4.

---

### C5 🟠 🔮 Renderer-specific material types

**What happens** — Materials come through as default grey.

**Why** — V-Ray and Corona have their own material types with their own settings:
`VRayMtl`, `VRayBlendMtl`, `CoronaMtl`, `VRayLightMtl`, plus multi/sub-object
materials that hold several materials in one slot. FBX carries almost none of
this properly.

**Found in** — Predicted. **No V-Ray or Corona file has ever been tested.** This
is listed as a critical unknown in the existing research notes and remains one.

**Fix** — The agency-side script reads these directly in 3ds Max, where they are
fully available, and writes out a plain description (which image in which slot,
which numbers). That is the entire reason the material description format exists
and why extraction happens on the agency's machine rather than ours.

Handle the common cases explicitly and let everything else fall through to
"needs baking":
- `VRayMtl` → straightforward mapping
- `VRayBlendMtl` → take the base layer, flag the rest
- Multi/sub-object → split by face assignment
- `VRayLightMtl` → emissive preset
- Falloff maps → usually approximated by a fixed roughness value

**How to catch it** — Count how many materials could not be read confidently and
report it as an automation-rate number. That percentage is directly a business
metric.

---

### C6 🟠 ❌ Ambient occlusion is packed but never used

**What happens** — Corners and crevices are not subtly darkened, and the bytes
carrying that information are shipped anyway.

**Why** — We correctly pack occlusion into the red channel of a combined image,
but no material tells the viewer to use it. Blender needs a specific node
arrangement to export occlusion, which is not wired up.

**Found in** — P0.4.

**Fix** — Wire the exporter's occlusion output, or write the occlusion reference
directly into the glTF file after export. Small, well-understood fix.

---

### C7 🟠 ⚠️ Brightness shift after processing — half found, half open

**What happens** — The processed model rendered uniformly **0.28 brighter** than
the original on every view. Similarity 0.73 where 0.90+ is wanted.

**Why — the larger half, now found and fixed.** Not a material bug at all. The
artist's file carries a saved **exposure setting of −1.0** — one stop darker,
i.e. half brightness. The baseline render loads that file and inherits it; the
quality-check render starts from a blank Blender and does not. The two sides
were rendered at different exposures and the comparison measured that.

**The clue was the sky.** The sky is background — no model material touches it.
Once the *sky* was also brighter, a material bug was impossible and only a
global setting could explain it. Worth remembering as a debugging technique:
**find the part of the image the suspected cause cannot possibly affect.**

Fixing it moved similarity from **0.7324 to 0.8738** and the offset from +0.281
to +0.156.

**Why it went unnoticed** — Only `view_transform` was being pinned. Every other
colour setting was inherited from whatever file was open.

**Found in** — P0.4.

**Fix (applied)** — `pin_color_management()` sets view transform, look, exposure,
gamma, curve mapping, display device and film transparency explicitly on both
paths.

> **General rule:** a comparison renderer must **control** every setting that
> affects output pixels. Anything left unset is inherited, and the comparison
> then measures the file rather than the pipeline.

**Still open** — the remaining +0.156. Consistent with C4 (unbaked procedural
ground) and C6 (unbound occlusion): the ground-dominated top view is still the
worst at 0.8365. Fix those two before investigating further.

---

### C8 🟠 ✅ The recommended texture compression costs four times the budget

**What happens** — Everything works, and the file is 18.5 MB against a 5 MB
budget.

**Why** — The research recommends a high-quality compression mode (UASTC) for
normal maps, because compression artifacts in them are very visible. That is
true. It is also roughly four times larger than the alternative, and normal maps
are the majority of our images.

**Found in** — P0.4.

| setting | file size | budget |
|---|---|---|
| Uncompressed | 29.96 MB | 6× over |
| **Recommended (UASTC normals)** | **18.51 MB** | **~4× over** |
| ETC1S everywhere | 4.51 MB | fits |
| **ETC1S, normals kept larger** | **4.23 MB** | **fits** |

**Fix** — Use the cheaper mode everywhere, but keep normal maps at higher
*resolution* to compensate. Revisit if a client complains about surface quality.

**General lesson** — **Quality recommendations written without a byte budget are
not usable as-is.** Every such recommendation needs re-measuring against the
actual constraint.

---

### C9 🟡 🔮 Combining maps of different sizes or different UV layouts

**What happens** — The combined occlusion/roughness/metallic image is subtly
wrong — roughness misaligned with the surface.

**Why** — Packing three images into one assumes all three line up. If the
roughness map is 2048 and the occlusion map is 512, or if they were made for
different UV layouts, they do not.

**Found in** — Predicted. Our test set happened to be uniform.

**Fix** — Resize all three to a common size before combining (already done), and
**check they use the same UV channel** before packing. If they do not, do not
pack — ship them separately and report why.

---

### C10 🟡 🔮 Normal maps with the green channel inverted

**What happens** — Surfaces look lit from the opposite direction. Bumps look like
dents.

**Why** — Two conventions exist (OpenGL and DirectX) and they differ by flipping
one channel. 3ds Max commonly uses the DirectX convention; glTF requires OpenGL.

**Found in** — Avoided by choice in P0.4 (we deliberately downloaded the correct
convention), but **not detected or corrected** for arbitrary input.

**Fix** — Flip the green channel when the source is known to be DirectX. Detecting
it automatically is genuinely hard; the practical answer is to ask the artist once
per studio and record it as a per-client setting.

---

### C11 🟡 ❌ Texture tiling settings are not carried

**What happens** — A brick texture that should repeat every 2 m repeats once
across the whole wall, or a thousand times.

**Why** — In 3ds Max, tiling is a property of the material, not of the mesh.
Our material format does not currently record it.

**Fix** — Record the tiling values in the material description and translate them
to glTF's texture-transform extension, which supports exactly this.

---

### C12 🟠 ✅ Texel density varies wildly between objects

**What happens** — Bricks are tiny on one wall and enormous on the next.

**Why** — Measured on the test model, the size of one texture repeat ranged from
**0.35 m to 108,000 m** — five orders of magnitude. Artists set up mapping per
object, for the camera angles they cared about, and never for consistency.

**Found in** — P0.4.

**Fix** — Measure each object's texel density and scale its UVs to a target
real-world size. Scaling preserves the artist's seams; re-projecting would throw
their work away.

**Caveat** — This only works if density is roughly consistent *within* one object.
Three objects in our test had high internal variation, so a single scale factor is
only approximate for them. They are flagged rather than silently fudged.

---

### C13 🟡 ❌ Multiple UV channels with different meanings

**What happens** — The wrong texture coordinates are used and everything smears.

**Why** — Arch-viz models often carry two or three UV channels: one for the main
texture, one for a detail overlay, one for lightmaps. FBX carries them all with
unhelpful names.

**Fix** — Record which channel each texture uses in the material description.
Default to the first channel, but never assume it silently.

---

# PART D — Simplification and optimisation

### D1 🟠 ✅ Object names are useless for classification

**What happens** — Nothing is classified correctly, and everything gets treated
as a generic solid.

**Why** — The design assumed names like "wall", "floor", "railing" would identify
60–70% of objects. Measured on a real export: **0%**. 3ds Max names everything
`Box002`, `Cone001`, `Line005`. Artists have no reason to rename them.

**Found in** — P0.3.

**Fix** — Classify from geometry alone. Names are not consulted at all now.

**General lesson** — A rule that never fires but *looks* authoritative in the
code is worse than no rule, because it makes you believe the problem is handled.

---

### D2 🟠 ✅ The obvious shape metric measures nothing

**What happens** — Classification puts almost everything in one bucket.

**Why** — "Flatness" (thinnest dimension ÷ longest) sounds like a good way to
find walls and floors. On this model nearly every object scored 0.25–0.41.

The reason is worth understanding: these objects are **building-spanning shells**.
A single "facade" object wraps the entire tower, so its bounding box *is* the
building. It tells you nothing about whether the object is made of thin slats or
big flat panels.

**Found in** — P0.4.

**Fix** — Measure the *pieces*, not the object:

| measurement | tells you |
|---|---|
| triangles per m² | how finely detailed |
| m² per piece | how big each element is |
| triangles per piece | whether it is already as simple as possible |

These cleanly separate louvres (2287 triangles/m²) from partition walls (1.8).

---

### D3 🔴 ✅ An object was simplified out of existence

**What happens** — The paving slab the building stands on **completely
disappeared** from the output.

**Why** — It was a 12-triangle slab. Its class was allowed to reduce to 4%. Four
percent of twelve is zero.

> A ratio is meaningless on geometry that is already minimal. 4% of a box is not
> a smaller box — it is nothing.

**Found in** — P0.4.

**Fix** — Two absolute limits regardless of any percentage: never reduce below 64
triangles, and never attempt aggressive simplification on anything under about
1,200 triangles.

**How to catch it** — After simplification, **assert that every object that
existed before still exists and still has triangles.** Trivially cheap, and it
would have caught this instantly.

---

### D4 🔴 ✅ The recommended reduction ratios destroy lean models

**What happens** — The facade comes out jagged and torn, with visible chunks
missing.

**Why** — The research specifies keeping 5–15% of triangles for walls. Those
numbers assume a dense, heavily subdivided mesh. **Our model is already efficient
box geometry**, so the same ratio deletes it rather than simplifying it.

**Found in** — P0.4.

**Fix** — Raised the limits substantially (walls from 5% to 22%, solids from 15%
to 35%). More importantly, the *general* fix: the ratio must depend on how dense
the mesh already is, not be a fixed number per class. A wall at 50 triangles/m²
can afford 10%; a wall at 2 triangles/m² cannot afford 50%.

**General lesson** — **Ratios are the wrong unit.** The right target is triangles
per square metre of surface, which is comparable across models. This is a
worthwhile redesign.

---

### D5 🟠 ✅ Thin things shatter under standard simplification

**What happens** — Railings, louvres and window frames turn into mangled spikes.

**Why** — The usual simplification method repeatedly picks the "least important"
edge and collapses it. On a 37 mm baluster, every edge is important.

**Found in** — P0.2.

**Fix** — For thin repeated detail, use *planar dissolve* instead — which merges
faces that are already flat and in line. It is **lossless**: the shape is
genuinely unchanged. Measured, it took one louvre bank from 115,872 to 37,536
triangles **with no shape change at all**, beating the target the lossy method
was aiming for.

---

### D6 🟠 ❌ UV seams block lossless simplification

**What happens** — Planar dissolve reports success but removes almost nothing on
some objects.

**Why** — Dissolve is told not to merge across texture boundaries, to avoid
smearing. But box-projected UVs create a boundary on *every* face, so nothing can
merge.

**Found in** — P0.4 (observed; not yet fixed).

**Fix** — When we generated the UVs ourselves, we know they are box-projected and
can be regenerated. In that case, dissolve first and generate UVs afterwards.
Reversing those two steps is likely a significant free win.

---

### D7 🟠 ⚠️ The triangle target is not reachable, on two different models

**What happens** — Simplification stops short of the target and reports a miss.

**Why** — Different reasons each time, which is instructive:
- P0.2: 83% of the scene was thin louvre geometry that shatters under
  simplification. It stopped at 41% kept.
- P0.4: much of the scene is already minimal box geometry with nothing to remove.
  It stopped at 149,624 against a 120,000 target.

**Fix** — Both cases genuinely need **removing whole elements** or **building
proper levels of detail**, not simplifying harder. Simplifying harder is exactly
what produced D3 and D4.

Realistic approach: at distance, replace repeated fine detail (a bank of 400
louvres) with a single flat surface carrying a picture of louvres. This is the
standard game-industry answer and it is not implemented.

**Credit where due** — The pipeline reports the miss honestly rather than faking
the number. That is correct behaviour and should be preserved.

---

### D8 🔴 ⚠️ Draw calls cannot be reduced by merging meshes

**What happens** — After merging every object per floor, draw calls fell from
around 174 to 160 — nowhere near the target of 100.

**Why** — A "draw call" is one instruction to the graphics card. You need a
separate one **per material**, no matter how you group the meshes. Twelve storeys
× thirteen distinct materials = 160, and no amount of mesh merging changes that.

**Found in** — P0.3.

**Fix** — Reduce the number of *materials*, not meshes. Combine many textures into
one large sheet (an atlas) so many materials become one.

**The catch** — Atlasing breaks tiling. A wall using a texture repeated 20 times
cannot be atlased without baking that repetition into the image, which makes it
enormous. So: **atlas only non-tiling materials**, and merge single-colour
materials into a small palette image. Arch-viz scenes are full of flat colour
materials, so this alone should help a lot.

**Note** — P0.4 measured only 17 draw calls, but that is one building at one level
of detail. The problem returns at full scale.

---

### D9 🟡 ❌ Smoothing information is lost or misapplied

**What happens** — Curved surfaces become faceted, or flat surfaces get wavy
shading.

**Why** — 3ds Max stores "smoothing groups"; Blender stores "custom split
normals"; glTF stores per-vertex normals. Information is lost at each hop, and
simplification invalidates what survives.

**Fix** — Preserve custom normals through import (currently done), and
**recalculate them after simplification** with an angle threshold rather than
inheriting stale data.

---

# PART E — Output, delivery and viewer

### E1 🔴 ✅ Lighting was skipped because one lamp existed

**What happens** — All interiors render pure black.

**Why** — The check was "does this scene have any lights?" It found one sun, said
yes, and skipped the whole lighting setup. But a sun with no sky lights only what
it can see directly. Everything in shadow got nothing.

> A sun without a sky is not lighting.

**Found in** — P0.3.

**Fix** — Build the sky and the sun **together**, from one shared set of angles,
independent of whether lamps already exist. Otherwise the horizon says sunset and
the shadows say midday.

---

### E2 🔴 ✅ Windows were opaque, and then still dark

**What happens** — You cannot see through the windows. Then, after fixing the
glass, interiors are *still* black.

**Why** — Two separate problems stacked. First, transparency was set up in the
source scene but FBX did not carry it. Second, glass that the camera can see
through but the *shadow calculation* cannot is exactly as dark as a brick wall.

**Found in** — P0.3.

**Fix** — Apply a glass preset from the agency's tag, **and** stop glass casting
shadows.

---

### E3 🟠 ✅ Realistic glass is 20× too slow

**What happens** — Rendering time goes from 47 seconds for sixteen viewpoints to
3 minutes 45 for two.

**Why** — Physically correct transparency forces the renderer to re-draw the
whole scene to work out what is behind the glass. In the browser it nearly
doubled draw calls too — on the level already over budget.

**Found in** — P0.3.

**Fix** — Use simple transparency instead. It achieves the actual goal (daylight
in the rooms, you can see through the window) at a fraction of the cost. Strip
the expensive setting before export.

**General lesson** — **Physical correctness is not the goal. Looking right within
budget is the goal.**

---

### E4 🟠 ✅ The viewer's environment was the fidelity bottleneck

**What happens** — Diagonal bands across the facade; every downward-facing
surface nearly black.

**Why** — Most of the building was untextured flat grey, which means **what you
see on it is almost entirely the environment reflection**. The viewer used a
tiny two-stop gradient as both backdrop and lighting.

**Found in** — P0.2.

**Fix** — Use a proper procedural room environment (about 5 KB, no downloaded
asset). Cheap fix, large visual improvement.

**General lesson** — On an untextured or lightly-textured model, **lighting *is*
the material**. Effort spent on the environment pays back more than effort spent
on geometry.

---

### E5 🔴 ❌ Deep links can silently point at the wrong room

**What happens** — A QR code printed on a hoarding stops working, or worse, opens
a different flat than it did last month.

**Why** — Hotspot identifiers are currently positional — `f0_h0` means "the first
one found on floor zero". Re-process the model after any change and the ordering
can shift.

**Found in** — Identified in P0.3, **not fixed**.

**Fix** — Generate identifiers from stable content (the room's position in the
building, rounded sensibly) rather than from discovery order. A printed URL is a
public promise.

---

### E6 🔴 🔮 Memory leaks crash phones

**What happens** — The viewer works fine, then the browser tab dies after a few
minutes of exploring.

**Why** — Moving between levels loads new geometry and textures. If the old ones
are not explicitly released, memory grows until the phone gives up. Phones have
far less headroom than desktops and simply kill the tab.

**Found in** — Predicted, and designed for but never verified.

**Fix** — Explicitly release geometry, materials and textures on every level exit.
**Verify with a memory snapshot after ten navigation cycles** — steady growth
means a leak.

---

### E7 🟠 🔮 A missing decoder shows an empty scene with no error

**What happens** — Blank screen. Console shows nothing wrong.

**Why** — Compressed geometry and textures need small decoder programs loaded
alongside the viewer. If the path is wrong, loading fails quietly.

**Fix** — Check the decoder loaded before doing anything else, and show a real
error if not. **Always verify this before debugging any "model not appearing"
problem** — it is the most common cause and the least obvious.

---

### E8 🔵 ❌ We cannot know which rooms make up a flat

**What happens** — The viewer wants to show "Flat 3B". We can only show "room",
"room", "room".

**Why** — Room detection finds *rooms*. A saleable flat is a **business grouping**
of rooms. Two identical 2BHKs on the same floor are geometrically indistinguishable
— the difference lives in the price list, not the model.

**Found in** — Identified in P0.3, still open.

**Fix** — **The pipeline cannot derive this and must not guess.** The agency must
supply it, as a simple list in the intake package. This is a product and
onboarding problem, not a technical one, and pretending otherwise will produce
confidently wrong flat numbering.

---

### E9 🔵 ❌ Sale status must never be baked into the model

**What happens** — Marking one flat as sold requires reprocessing an entire
building.

**Why** — Availability changes daily; the model changes rarely.

**Fix** — Keep availability entirely outside the 3D files, fetched separately at
view time. Already decided, but nothing enforces it yet.

---

### E10 🟡 ❌ Transparent objects draw in the wrong order

**What happens** — Looking through a glazed balcony at a window behind it, one of
them disappears or draws in front.

**Why** — This is a genuine, unsolved limitation of how browsers draw
transparency. There is no clean fix.

**Fix** — Mitigate, do not solve: keep glass to a single layer where possible,
sort transparent objects back-to-front, and accept the remaining cases. Worth
documenting for client expectations rather than chasing.

---

# PART F — Pipeline architecture and process

### F1 🔴 ✅ Blender hung indefinitely using no CPU

**What happens** — A run sat for 12 minutes having used 2.49 seconds of
processing. Nothing in the log.

**Why** — Loading the artist's file with their add-ons enabled started an add-on
that tries to open a network connection, which is impossible in headless mode. It
waited forever.

**Found in** — P0.4.

**Fix** — Always launch with add-ons and preferences disabled. This is already a
written project rule; it is now proven to be the difference between working and
hanging.

**Also** — Put a **hard timeout** on every stage. A stage that has produced no
output for ten minutes should be killed and reported, not waited on forever.

---

### F2 🟠 ✅ Stage output was invisible until the end

**What happens** — You cannot tell whether a long run is working or stuck.

**Why** — Print output was buffered and only appeared when the program finished —
which, in F1's case, was never.

**Fix** — Flush every progress message immediately, and print a line as each stage
starts. Costs nothing, and turned an unlocatable hang into a five-second
diagnosis.

---

### F3 🟠 ❌ Every run redoes everything

**What happens** — Changing one material colour costs a full reprocess.

**Why** — There is no caching. Each run starts from nothing.

**Fix** — Key each stage's output by a fingerprint of its inputs and settings. A
re-run after a material tweak then reuses the geometry work. This turns "client
wants the sofa in blue" from a multi-hour rebuild into a few minutes.

**Why it matters** — This is a **margin** issue, not a convenience one. Client
revision requests are the normal case, not the exception.

---

### F4 🟠 ❌ Nothing runs in parallel

**What happens** — Processing takes longer than it needs to.

**Why** — Everything is sequential. Texture conversion in particular is
embarrassingly parallel — 40 independent images processed one at a time.

**Fix** — Process textures in parallel. Render viewpoints in parallel. Both are
straightforward and neither touches the sequential geometry work.

---

### F5 🟠 ⚠️ Reported numbers were not measured

**What happens** — A report claims a 93.4% reduction that was really 90.0%.

**Why** — Sizes were added up per material, so textures shared between materials
were counted several times over. Since the test model shares 11 texture sets
across 16 materials, this inflated the "before" figure by about 53%.

**Found in** — P0.4.

**Fix** — Always count unique files. More generally: **measure outputs by
inspecting them, never by adding up what you think you did.**

---

### F6 🟠 ✅ The same logic was written three times, differently

**What happens** — Two different stages disagreed about which objects were the
building, and both were wrong in different ways.

**Why** — Each stage worked it out for itself.

**Found in** — P0.4.

**Fix** — Derive it once, record it on the object, and read it everywhere else.

**General rule** — If two stages need the same judgement, it belongs in one shared
place. This applies next to: which objects are glass, which are context, and which
level of detail each belongs to.

---

### F7 🟠 ✅ A test fixture too clean to test anything

**What happens** — The material classifier reported 100% "simple" — no complex
cases at all — because the preparation step had rebuilt every material as a clean
one.

**Why** — I scrubbed away exactly the complexity the classifier exists to handle.

**Found in** — P0.4.

**Fix** — Deliberately preserve the messy cases. The test fixture was changed to
restore the bump-wired and procedural materials the source scene really had,
giving a realistic 82/12/6 split.

**General lesson** — **A clean fixture proves nothing.** The test set needs to
contain, on purpose: a missing texture, a non-English name, wrong units, a
mirrored object, a proxy, a procedural material, and an unwelded mesh.

---

### F8 🔴 ❌ No real client file has ever been processed

**What happens** — Every number and threshold in the project is tuned against
one stock model.

**Why** — No V-Ray or Corona file has been obtained. This has been the stated top
blocker since P0.2 and it is still true.

**Fix** — Acquire **at minimum**: one 3ds Max + V-Ray scene, one 3ds Max + Corona
scene, and one SketchUp model, ideally with genuine interiors. Run all of them
before trusting any tuned number.

**This is the single highest-value action available and it is not a coding task.**

---

### F9 🟡 ❌ No automated quality gate actually gates anything

**What happens** — A bad result would ship.

**Why** — The comparison exists and produces numbers, but nothing acts on them,
and one of the numbers (C7) is not trustworthy.

**Fix** — Once C7 is understood, set real thresholds and make the pipeline exit
with a failure when they are breached. Route failures to a human review queue
with the specific failing view attached.

**Track the percentage of jobs that pass with zero human involvement.** That
number is directly the gross margin of the business.

---

### F10 🟡 ❌ The official glTF validator is not wired in

**What happens** — Structurally invalid output could ship undetected.

**Fix** — Run the standard Khronos validator on every output file and fail on
errors. It is a single command and catches a whole class of problems for
essentially no effort.

---

### F11 🟡 ❌ No progress reporting for the agency

**What happens** — The agency uploads 250 MB and sees nothing for ten minutes.

**Fix** — Emit stage-level progress that a future dashboard can display. The
report format already carries the timings; nothing surfaces them live.

---

### F12 🟡 🔮 No limit on input size

**What happens** — A 4 GB upload exhausts the server.

**Fix** — Cap the package size, cap the triangle count (already audited), cap the
texture count, and reject with a clear message rather than dying mid-run.

---

# PART G — Concept-level gaps

These are not bugs. They are decisions that are wrong, missing, or being avoided.

### G1 🔵 Which way do panoramas get made?

The project documents contradict each other: one says the agency renders
photo-real panoramas in Corona and they never enter our pipeline; the other says
we generate them ourselves. Both are supported, since the format points at a
folder of images either way — **but nobody has decided which is the default.**

This matters because it determines whether interior quality is "as good as the
agency's renderer" or "as good as ours", which is a sales argument, not a
technical detail.

---

### G2 🔵 What does "good enough" mean?

There is no agreed quality threshold. Without one, "is this ready to ship" is a
matter of opinion, every job needs a human, and the business does not scale.

**Fix** — Pick numeric thresholds, even arbitrary ones, and tighten them with
experience. A wrong threshold that is written down is far more useful than a
right instinct that is not.

---

### G3 🔵 Re-uploads and versioning are undesigned

When a client sends v2 of a building, does the old URL update or does a new one
appear? What happens to a QR code already printed on a hoarding? Is there a
rollback?

Deferred, but **it must not be designed into a corner**. The content-addressed
package format helps, since it makes two versions genuinely comparable.

---

### G4 🔵 There is no repair path for failures

The design says "no blocking human step", which is right as a default. But
something must happen to the 10–20% of jobs that fail a quality gate. Currently
they would simply fail.

**Fix** — A small review tool showing the failing view, the difference, and one
or two common corrections. Not a full editor. The goal is to move a job from
"failed" to "shipped" in two minutes, not to let a human redo the work.

---

### G5 🔵 The 3ds Max side does not exist yet

Everything agency-side is currently written in Blender as a stand-in. The rules
have been kept implementable in 3ds Max, and the equivalences are documented, but
**not one line of MaxScript has been written or tested**.

Risks that will only appear then: 3ds Max's scripting is slow on large scenes;
version differences between Max 2018 and 2024 are significant; and reading
material properties requires branching on type rather than assuming property
names exist.

---

### G6 🔵 SketchUp is entirely untouched

Designed for, never built. The main known difference is that SketchUp models
carry front/back face information that is genuinely useful for fixing normals
(B2), and that its export path is direct to glTF rather than through FBX.

---

# PART H — What to do first

Ordered by value divided by effort.

### Do immediately (days, high value)

1. **F8 — Get real client files.** Nothing else on this list is trustworthy
   until this happens. Not a coding task.
2. **D3/F5 assertions — make every stage prove it did its job.** After
   simplification, assert every object still exists. After packaging, reopen and
   verify. After each stage, measure rather than predict. This converts the
   project's single most common failure mode into a loud error.
3. **C7 — ~~find the brightness shift~~ DONE (exposure inheritance).** Similarity
   0.7324 → 0.8738. Remainder now depends on C4 and C6 rather than being a
   mystery.
4. **A6 — sanity-check units against real building dimensions.** Cheap, and
   catches an expected-40% input problem.
5. **F10 — wire in the glTF validator.** One command.
6. **C6 — bind the occlusion texture.** Small, known fix, already-paid-for bytes.

### Do next (weeks)

7. **C4 — implement baking** for procedural materials. Largest known visual gap.
8. **D6 — reorder dissolve and UV generation.** Likely a large free reduction.
9. **D8 — material atlasing.** The only route to the draw-call budget.
10. **E5 — stable identifiers.** Blocks the first public URL.
11. **F3 — stage caching.** Directly a margin issue on revisions.
12. **F7 — build a deliberately nasty test fixture.**

### Do when the relevant version arrives

13. **B2 — normal repair** (needed the moment a CAD or SketchUp file arrives).
14. **C5 — V-Ray and Corona material handling** (needed with F8).
15. **E6 — memory leak verification** (needed before any public launch).
16. **D7 — level-of-detail generation** for repeated fine detail.

---

## Closing observation

Almost every problem in this document falls into one of three groups:

1. **The input is messier than the design assumed.** Every version has found new
   ways for this to be true, and real client files will find more.
2. **A stage did nothing, or did the wrong thing, and said it succeeded.** This
   is the expensive category, because the symptom appears far from the cause.
3. **A written recommendation was wrong once measured.** The UV transfer
   technique, the compression mode, the reduction ratios, the name-based
   classification — all were reasonable-sounding and all were wrong in practice.

The defence against the first is a genuinely nasty test fixture. Against the
second, per-stage assertions. Against the third, measuring before trusting —
including measuring whether a "fix" actually improved anything, since at least
one of ours made things dramatically worse while appearing to work.
