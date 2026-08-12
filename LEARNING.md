# LEARNING.md — the 3D vocabulary of this project

Every technical concept this project uses, explained from zero. Written for
someone who can read code but has never worked in 3D.

**Depth follows importance.** Concepts that caused real bugs here get pages;
concepts you only need to recognise get a paragraph. Each section is tagged:

- 🔴 **Core** — you cannot follow this project without it
- 🟡 **Working** — comes up regularly; understand it well enough to reason about
- ⚪ **Recognise** — know what the word means when you see it

At the end, [Part 15](#part-15--where-each-concept-actually-bit-us) maps every real
bug in this project back to the concept behind it. If you only read one section,
read that one after skimming the rest.

---

## Contents

1. [Mesh anatomy](#part-1--mesh-anatomy) 🔴
2. [Space, transforms and units](#part-2--space-transforms-and-units) 🔴
3. [Normals and shading](#part-3--normals-and-shading) 🔴
4. [UVs and texture mapping](#part-4--uvs-and-texture-mapping) 🔴
5. [Materials and PBR](#part-5--materials-and-pbr) 🔴
6. [Colour spaces](#part-6--colour-spaces) 🔴
7. [File formats](#part-7--file-formats) 🟡
8. [Geometry reduction](#part-8--geometry-reduction) 🔴
9. [Light, rendering and tone](#part-9--light-rendering-and-tone) 🟡
10. [Cameras, projection and depth](#part-10--cameras-projection-and-depth) 🟡
11. [Real-time and web delivery](#part-11--real-time-and-web-delivery) 🟡
12. [The detection algorithms](#part-12--the-detection-algorithms) 🟡
13. [Blender vocabulary](#part-13--blender-vocabulary) 🟡
14. [Arch-viz domain words](#part-14--arch-viz-domain-words) ⚪
15. [Where each concept actually bit us](#part-15--where-each-concept-actually-bit-us)

---

# Part 1 — Mesh anatomy
🔴 **Core**

## The three primitives

A 3D model is not a solid object. It is a **hollow shell** — an infinitely thin
surface with nothing inside. A 3D building is a stage set, not a building.

That shell is built from three things:

**Vertex** (plural **vertices**) — a point in space, just `(x, y, z)`. The corners.

**Edge** — a line joining two vertices.

**Face** (or **polygon**) — a flat surface enclosed by three or more edges. This is
what you actually see.

```
      v3────────v2        4 vertices
      │          │        4 edges
      │   face   │        1 face
      │          │
      v0────────v1
```

**Mesh** — the whole collection of vertices, edges and faces making one object.

## Why triangles are special

Faces come in three flavours:

| Name | Corners | Notes |
|---|---|---|
| **Triangle** (tri) | 3 | Always perfectly flat — three points define a plane |
| **Quad** | 4 | The modelling favourite; can be non-planar |
| **N-gon** | 5+ | Convenient to model with, unpredictable to render |

**Graphics cards can only draw triangles.** Everything else gets chopped into
triangles before rendering. This is why every performance number in this project
is in triangles, never faces.

> ### 🔴 The n-gon counting trap
>
> An n-gon with 6 corners is **one face** but **four triangles**. Code that counts
> `len(mesh.polygons)` to estimate GPU cost is wrong, potentially by several times.
>
> This project always counts triangles via evaluated `loop_triangles`, never
> polygons. See [Part 13](#depsgraph).

## Topology

**Topology** = how the vertices connect, as distinct from where they are.

Two meshes can have identical shape and completely different topology — the same
wall could be 2 triangles or 2,000. Topology quality determines whether decimation
works, whether shading looks right, and whether UVs behave.

## Islands and connected components

**Island** (or **loose part**) — a chunk of mesh not connected to the rest by any
edge. A balcony railing with 200 separate balusters is **one object with 200
islands**.

Islands are how you tell object *kinds* apart without names. A railing is
"thousands of small disconnected parts"; a wall is "one big connected sheet".

> ### 🔴 This project's most important discovery
>
> A 3ds Max FBX export arrives with meshes **exploded into disconnected
> triangles** — every single triangle is its own island, sharing no vertices with
> its neighbours.
>
> Measured: one object had **115,872 triangles across 115,872 islands**, using
> 347,616 vertices where 58,752 would do. Each triangle carried its own private
> three corners sitting in exactly the same place as its neighbours'.
>
> Two things break silently:
> 1. Decimation does nothing — it works by merging vertices across **shared**
>    edges, and there are none.
> 2. Every island-based heuristic reads garbage — triangles-per-island is `1` for
>    the whole scene.
>
> The fix is welding (below), and it must happen before anything else.

## Welding

**Weld** (also **merge by distance** or **remove doubles**) — find vertices closer
together than some threshold and fuse them into one.

This is what converts 115,872 loose triangles into a connected surface. This
project uses `1e-4` (0.1 mm): tight enough to leave genuinely separate parts
separate, loose enough to close floating-point noise from the FBX exporter.

Cost: about **0.9% of triangles**. Value: everything downstream.

## Degenerate geometry

**Degenerate face** — a face with zero area. Three corners in a straight line, or
two corners in the same place. Invisible, but it confuses every algorithm that
divides by area and wastes memory.

**Loose vertex** — a vertex belonging to no face. Invisible, still costs memory.

**Non-manifold** — geometry that couldn't exist in the real world: an edge shared
by three faces, or faces meeting only at a point. Breaks operations that assume a
sensible surface.

## Sliver triangles

**Sliver** — a long, thin, needle-like triangle. Measured by **aspect ratio**:
longest edge ÷ shortest edge.

```
   good (≈1:1)              sliver (68:1)
      ╱╲              ╱────────────────────────╲
     ╱  ╲             ╲────────────────────────╱
    ╱____╲
```

Slivers are legal but cause trouble: shading interpolates badly across them,
numerical operations lose precision, and they often signal careless triangulation.

> ### 🔴 Measured in this project
>
> | Object | Faces | Slivers (>20:1) | Worst |
> |---|---|---|---|
> | `Cone001` | 115,872 | **97,920 (85%)** | 68:1 |
> | `Box002` | 7,704 | 5,136 (67%) | **143:1** |
>
> 3ds Max exports pre-triangulated using a **fan** pattern — pick one corner,
> connect it to every other. Fast, but on a long rectangle it produces exactly
> these needles. Combined with smooth shading ([Part 3](#part-3--normals-and-shading))
> this is the cause of the diagonal streaking visible on the facade.

## Coplanar faces and redundancy

**Coplanar** — lying in the same flat plane.

If a flat wall is made of 500 triangles that all lie in one plane, 498 of them
describe *nothing*. You could delete them and the shape would be identical.

This project measured **44% of all triangles as redundant coplanar subdivision** —
191,310 down to 107,956 with no vertex moved at all. See
[planar dissolve](#planar-dissolve-lossless).

## Bounding box

**Bounding box** (usually **AABB**, axis-aligned bounding box) — the smallest
box, aligned to the X/Y/Z axes, that contains an object.

Cheap to compute, and enormously useful. This project uses it for: object size,
flatness detection, building footprint, camera framing, and validating that the
exported file matches the scene.

**Flatness** = `min(dimensions) / max(dimensions)`. Below 0.05, the object is
essentially a plane — which is how walls and floors are detected without names.

---

# Part 2 — Space, transforms and units
🔴 **Core**

## Local vs world space

Every object stores its geometry in **local space** — coordinates relative to its
own origin. The object then carries a **transform** placing it in the scene.

```
local space                    world space
vertex at (0, 0, 1)     +      object at (50, 30, 0)    =   (50, 30, 1)
                               rotated 90°, scaled 2×        (after maths)
```

**This distinction causes more silent bugs than anything else in 3D.**

Local-space values are meaningless for measurement. An object scaled to 0.0254 has
local coordinates 39× larger than its real size. Blender's `polygon.area` and
`polygon.normal` are **local-space** and quietly wrong for any scaled object —
which in a 3ds Max export means *every* object.

This project transforms everything to world space before measuring anything.

## The transform matrix and TRS

An object's placement is a 4×4 **matrix** combining three things, always applied in
this order:

**T**ranslate (move) · **R**otate · **S**cale — collectively **TRS**.

A matrix is just a compact way to store "move here, turn this way, resize by this
much" so it can be applied to thousands of vertices with one operation and combined
with a parent's transform by multiplication.

## Transforming normals is different

This is unintuitive and matters.

Under **non-uniform scale** (different factors per axis), normals do **not**
transform like positions. Squash a sphere flat and its surface normals must tilt
the *opposite* way to stay perpendicular to the surface.

The correct transform is the **inverse transpose** of the upper-left 3×3 of the
matrix. Using the plain matrix misclassifies tilted faces on any scaled object —
which would break this project's floor-vs-wall detection.

## Up axis — Z-up vs Y-up

Different software disagrees about which direction is "up".

| Convention | Used by |
|---|---|
| **Z-up** | Blender, 3ds Max, AutoCAD, most CAD and architecture |
| **Y-up** | glTF, three.js, Unity, most games and web |

Converting is a −90° rotation about X:

```
blender_to_gltf: (x, y, z) → (x, z, -y)
gltf_to_blender: (x, y, z) → (x, -z, y)
```

Get it wrong and **every model lies on its side**. Nothing errors. It looks perfect
in Blender.

> ### 🔴 Rule this project enforces
>
> The conversion lives in exactly **one module**, called from everywhere that
> crosses the boundary. Doing it inline in two places is how markers end up
> floating in plausible-but-wrong positions with no error anywhere.
>
> Note also: **a conversion module only rotates — it cannot absorb a scale error.**
> Scale agreement is asserted separately.

## Units

3D files carry no inherent unit. A "1" might be 1 metre, 1 centimetre or 1 inch.

**glTF is always metres**, by specification. Everything else varies. 3ds Max
commonly uses inches or centimetres. Roughly 40% of real submissions arrive with
wrong units.

> ### 🟡 In this project
>
> The test FBX is in **inches**. The importer handled it by putting a `0.0254`
> scale on each object's node transform rather than baking it into the vertices.
>
> This caused a false alarm: a validation check read the raw vertex bounds
> (mesh-local, in inches) **without applying the node transform** and reported the
> building as 75× oversized. A real investigation was launched into a scale bug
> that did not exist.
>
> **Lesson: any bounding-box check on a glTF file must walk the node hierarchy.**

## Origin and pivot

**Origin** — an object's own zero point. Rotation and scaling happen around it.

If a door's origin sits at its hinge, rotating it swings the door. If the origin is
at the world centre 50 m away, rotating it flings the door across the site.
"Unreset transforms" and "wrong pivot" are common defects in agency files.

## Floating-point precision

Coordinates are stored as 32-bit floats. These have **relative** precision — about
7 significant digits — so accuracy degrades as numbers grow.

At 10 metres you have sub-millimetre precision. At 10,000 metres you have
centimetres, and surfaces visibly jitter and flicker.

Some agencies model at real GPS coordinates, tens of thousands of units from the
origin. This project flags any object beyond a threshold distance.

---

# Part 3 — Normals and shading
🔴 **Core**

This is the part that caused the visible streaking on the facade, and it's the
least intuitive area in the whole document.

## What a normal is

A **normal** is an arrow perpendicular to a surface, indicating which way it faces.

```
        ↑ normal
   ─────────────  surface
```

Normals do two jobs:

1. **Which side is outside.** A face's normal points out of the solid.
2. **How light behaves.** Light hitting a surface head-on is bright; at a glancing
   angle, dim. That calculation uses the normal.

## Face normals vs vertex normals

**Face normal** — one arrow per face, computed from its corners. Unambiguous.

**Vertex normal** — an arrow stored *per vertex*, usually the average of the
adjacent faces' normals.

Rendering **interpolates vertex normals across the face**, blending smoothly
between the corners. That's the whole basis of smooth shading.

## Flat vs smooth shading

The same cylinder, same geometry, two different results:

```
FLAT shading                      SMOOTH shading
each face uses its own normal     normals interpolate across faces
you see every facet               looks curved
   ╱▔▔╲                              ╱▔▔╲
  ╱ ││ ╲   ← visible edges          ╱    ╲  ← seamless
```

**Neither is "correct".** Smooth is right for curved surfaces; flat is right for
things that genuinely have edges. A smooth-shaded cube looks like a melted
marshmallow.

## Auto-smooth (smoothing by angle)

The practical middle ground: **smooth where adjacent faces meet at a shallow angle,
flat where they meet sharply.**

With a 30° threshold: a cylinder's barrel (faces at ~5°) shades smooth, its end
caps (90°) stay crisp. One setting handles almost everything.

## Smoothing groups and custom split normals

3ds Max uses **smoothing groups** — numeric tags marking which faces should blend
together. Blender imports these as **custom split normals**: explicitly stored
per-face-corner normals that override the automatic calculation.

Custom normals are powerful and dangerous. They can encode exactly what the artist
intended — or preserve something nonsensical, immune to any smoothing setting you
apply on top.

> ### 🔴 The streaking bug, fully explained
>
> Measured on this project's asset: **every face of every object is marked smooth**,
> and every object carries custom split normals.
>
> Now combine that with the slivers from [Part 1](#sliver-triangles). Smooth shading
> interpolates the normal along the triangle. On a 68:1 sliver, that interpolation
> runs the full length of a needle — producing a visible gradient sweeping down the
> triangle's long axis.
>
> Across a facade tiled with fan-triangulated slivers, those gradients line up into
> **diagonal streaks**.
>
> The geometry is fine. The UVs are fine. The textures are fine. It's the normals.
>
> It is equally wrong in Blender — invisible only because Blender's *Solid* viewport
> mode ignores materials and uses simple studio lighting. Switch to Material
> Preview and it appears.
>
> **The fix:** clear the imported custom normals and apply auto-smooth at ~30°, so
> flat panels shade flat. Verified by before/after render.

## Winding order and backface culling

The order of a face's vertices — clockwise or counter-clockwise as seen from the
front — defines which side is the front. This is **winding order**.

**Backface culling** — an optimization where the renderer skips faces pointing
away, since you shouldn't see the inside of a solid object. Roughly halves the work.

**Double-sided** — culling disabled; the face renders from both sides, with the
normal flipped for the back. Necessary for architecture, where you legitimately
stand inside a wall shell and look at its back face.

> ### 🟡 In this project
>
> All 15 materials export as double-sided, so backfaces do render. When interiors
> looked dark, the cause was **lighting**, not backfaces — a downward-facing ceiling
> lit only from above is genuinely dark.

## Flipped normals

If normals point inward, the object renders inside-out — surfaces vanish from
outside and appear from within. A common defect. "Recalculate normals outside" is
the standard fix.

---

# Part 4 — UVs and texture mapping
🔴 **Core**

## The problem UVs solve

You have a 2D image. You have a 3D surface. How does the flat image wrap onto the
curved thing?

**UV mapping** is the answer: for every vertex of the 3D mesh, store a
corresponding `(u, v)` coordinate in the 2D image.

Think of a cardboard box. Cut along some edges, flatten it: that flat cross shape
is the **UV layout**. Print on the flat card, fold it back up, and the print lands
where you intended.

**Why "UV"?** X, Y and Z were taken. U and V are the next letters available.

## UV space

UV coordinates run 0 to 1 across the image, regardless of pixel size.

```
(0,1) ┌─────────┐ (1,1)
      │  image  │
      │         │
(0,0) └─────────┘ (1,0)
```

`(0.5, 0.5)` is always the centre, whether the texture is 256 px or 4096 px.

## Islands and seams

**UV island** — one connected patch in the flat layout.

**Seam** — where you cut the 3D surface to flatten it. Every cut is potentially a
visible line in the final render, so good unwrapping hides seams in creases and
corners.

Trade-off: fewer seams means more distortion; less distortion means more seams. You
cannot flatten a sphere without one or the other — the same reason every world map
distorts something.

## Tiling

UVs outside 0–1 make the texture **repeat**. A UV of 3.5 means "three and a half
times across the image".

This is how a small 1 m² brick texture covers a 30 m wall without being 30 m
across. Extremely common in architecture.

> ### 🔴 Tiling caused a spectacular bug
>
> This project's facade tiles **27–55×** (one object spans u from −13.39 to +14.39).
>
> The standard technique for preserving UVs through decimation is a "Data Transfer"
> pass that copies UVs from the original mesh to the simplified one by finding, for
> each new point, the nearest old face.
>
> **With tiling, that is incoherent.** Two neighbouring points on the new mesh can
> land on faces from *different tiles* — one at u≈2.1, one at u≈3.9. Interpolating
> between them smears the texture across the entire range.
>
> Measured UV-area change, with and without that pass:
>
> ```
> Line003    −16.0%   →   +21,311.5%
> Cone001     +0.0%   →      +222.0%
> ```
>
> The counter-measurement mattered just as much: Blender's own decimation
> interpolates UVs **perfectly well** — 0.0% area change on a 115,872-triangle
> object. The extra pass was solving a problem that did not exist.

## Texel density

**Texel** = texture pixel. **Texel density** = how many texture pixels land per
metre of real surface.

Consistent texel density across a scene is what makes it look coherent. If a wall
has 500 px/m and the floor beside it has 50 px/m, the floor looks blurry and cheap
even though both textures are "1K".

## Stretching and distortion

If UVs don't match the real proportions of the surface, the texture stretches. A
square of texture mapped to a long thin triangle smears along its length.

**Detecting it cheaply:** compute the total area covered in UV space (via the
**shoelace formula**, which gives a polygon's area from its corner coordinates) and
compare before and after an operation.

This project uses exactly that as a guard rail, because **triangle counts stay
perfectly plausible while the mapping underneath is destroyed**. UV area is the
cheapest signal that something broke.

## Unwrapping methods

**Smart UV Project** — automatic. Groups faces into islands by angle, flattens
each. The `angle_limit` parameter controls how aggressively faces merge into one
island: higher means fewer, larger islands and fewer seams. This project uses 66°;
Blender's default over-fragments architectural geometry badly.

**Lightmap Pack** — packs every face into non-overlapping space. Ugly to look at,
but guarantees each face has unique texture area, which is required for baking.

**Manual unwrapping** — an artist places seams by hand. Best quality, doesn't scale.

## Multiple UV channels

A mesh can carry several UV maps. In glTF these are `TEXCOORD_0`, `TEXCOORD_1`, etc.

Typical use: `TEXCOORD_0` for the visible material (tiled, overlapping is fine),
`TEXCOORD_1` for a baked lightmap (must be non-overlapping).

> ### 🔴 No UVs at all
>
> A mesh with no UV layer **cannot be textured**. There are no instructions for
> where the image goes. It renders as flat colour regardless of how good the
> material is.
>
> This project shipped exactly that bug: procedural unwrapping had been implemented
> inside the optimization stage, so turning optimization off silently exported two
> meshes with no `TEXCOORD_0`. Unwrapping moved to intake, where it belongs.

---

# Part 5 — Materials and PBR
🔴 **Core**

## Material vs texture

**Texture** — an image file.

**Material** — the full recipe for how a surface behaves: its colour, shininess,
bumpiness, transparency. A material *uses* textures, but also holds plain numeric
values.

## PBR — physically based rendering

The modern standard. Instead of inventing arbitrary parameters, PBR describes
surfaces the way physics does, so a material looks right under any lighting.

The dominant flavour, and the one glTF uses, is **metallic-roughness**:

| Channel | Meaning | Typical form |
|---|---|---|
| **Base Color** | The surface's inherent colour (also *albedo*, *diffuse*) | texture or RGB |
| **Metallic** | Is this metal? 0 = not, 1 = yes | usually 0 or 1 |
| **Roughness** | How blurry are reflections? 0 = mirror, 1 = chalk | value or texture |
| **Normal** | Fake surface bumpiness | texture |
| **Occlusion (AO)** | Where ambient light can't reach | texture |
| **Emission** | Light the surface emits itself | texture or RGB |

## Metallic is a hard switch, not a dial

This is the single most important material concept for this project, and the least
intuitive.

**Metallic is binary in reality.** A surface either is metal or isn't. Values
between 0 and 1 are physically meaningless — they exist only for blend maps.

The two behave completely differently:

| | Non-metal (dielectric) | Metal |
|---|---|---|
| Diffuse colour | **Yes** — the base colour you see | **None at all** |
| Reflection | Weak, white-ish | Strong, tinted by base colour |
| Appearance with no environment | Still shows its colour | **Black** |

> ### 🔴 The bug that made the building black
>
> A fully metallic surface has **no diffuse response whatsoever**. It is a mirror.
> Everything you see on it is reflected surroundings — so with nothing around to
> reflect, it renders black.
>
> Blender's FBX importer (the legacy one) set `Metallic = 1.0` on **all 15
> materials**, including "Wall Paint" and "Ceramic". In Blender's viewport this
> looked like plausible brushed metal, because Blender ships a studio HDRI for
> surfaces to reflect. Exported to the web with a weak environment, the building
> went black.
>
> The tell was uniformity: 15 materials sharing byte-identical
> `(metallic 1.0, roughness 0.859, specular 0.0)` is an importer default, not
> something an artist produced.

## Roughness vs glossiness

Two conventions for the same property, inverted:

- **Roughness** — 0 = mirror, 1 = completely diffuse. (glTF, Blender, Unreal)
- **Glossiness** — 1 = mirror, 0 = diffuse. (V-Ray, older pipelines)

`roughness = 1 − glossiness`. Getting this backwards makes everything shiny that
should be matte, and vice versa.

## Normal maps vs bump vs displacement

Three ways to add surface detail without geometry:

**Bump map** — greyscale. Bright = raised. Cheap, crude.

**Normal map** — RGB, where the three channels encode a direction vector rather
than a colour. This is why normal maps look lurid blue-purple: flat areas encode
"pointing straight out" as `(128, 128, 255)`. Much better than bump.

**Displacement map** — greyscale, but actually *moves* geometry. Real silhouette
changes; expensive; needs dense meshes.

Normal maps are the best value in real-time 3D. A flat wall with a brick normal map
is visually near-identical to modelled bricks — 200 triangles versus 50,000.

## Ambient occlusion

**AO** — a greyscale map darkening crevices, corners and contact points where
ambient light struggles to reach. Adds a lot of perceived depth cheaply.

## ORM packing

Occlusion, Roughness and Metallic are all **greyscale** — one number each. An RGB
image has three independent channels. So pack all three into one image:

```
AO        → Red channel
Roughness → Green channel
Metallic  → Blue channel
```

Five textures become three (BaseColor, Normal, ORM). **Zero quality loss** — the
data was one-dimensional anyway. 40% fewer downloads and 40% less GPU memory.

This is standard practice everywhere and **this project has not implemented it
yet.**

## Alpha and transparency

**Alpha** — a fourth channel for opacity. Three modes in glTF:

- **OPAQUE** — alpha ignored
- **MASK** — hard cutoff at a threshold; good for leaves and fences
- **BLEND** — true translucency; correct-looking but requires sorting and can
  produce artifacts

**Transmission** is different from alpha: it models light passing *through* glass
with refraction, rather than the surface simply being see-through.

## Node graphs

Modern materials are built as **node graphs** — small functional blocks wired
together. A texture feeds a colour-correction node, which feeds a mix node, which
feeds the shader's Base Color.

**BSDF** ("bidirectional scattering distribution function") is the maths describing
how light scatters off a surface. **Principled BSDF** is Blender's general-purpose
one, designed to match the PBR standard. glTF understands it directly, which is why
this project checks for it — any other setup gets approximated on export.

**VRayMtl** and **CoronaPhysicalMtl** are the proprietary equivalents in the
renderers agencies use. FBX cannot carry node graphs *at all* — the format predates
the concept — which is the central technical problem this whole product exists to
work around.

---

# Part 6 — Colour spaces
🔴 **Core**

Small topic, big consequences.

## The core idea

Human vision is non-linear. We distinguish dark tones far better than bright ones.
So image formats don't store brightness linearly — they store it **gamma-encoded**,
spending more bits on the darks where we can actually see the difference.

**sRGB** — the standard gamma encoding, roughly a 2.2 power curve. What JPEGs and
PNGs use. What a monitor expects.

**Linear** — raw physical light values. What rendering maths requires, because
light adds up linearly in reality.

Renderers convert sRGB → linear on load, do the maths, and convert back for display.

## The critical distinction

Some textures are **colour**. Some textures are **data that happens to be stored in
an image**.

| Texture | What it is | Colour space |
|---|---|---|
| Base Color | Colour | **sRGB** |
| Emission | Colour | **sRGB** |
| Roughness | A number 0–1 | **Non-Color / linear** |
| Metallic | A number 0–1 | **Non-Color** |
| Normal | A direction vector | **Non-Color** |
| AO | A number 0–1 | **Non-Color** |
| Displacement | A height | **Non-Color** |

Apply a gamma curve to a roughness map and every value is wrong — a surface meant
to be 0.5 rough reads as roughly 0.21. Everything comes out too shiny.

> ### 🔴 Why filename detection is a trap
>
> Blender's FBX importer marks **everything** sRGB. So data maps need correcting.
> The obvious approach — read the filename for `_Roughness`, `_Normal`,
> `_METALNESS` — is what most pipelines do.
>
> **It corrupted most of this building.**
>
> Five of seven textures in the test asset are named `*_AmbientOcclusion`,
> `*_Displacement` and `*_METALNESS` — and **every one is wired into Base Color**.
> The artist dragged whatever map looked right into the diffuse slot. That is the
> dominant real-world workflow.
>
> **The filename describes what the map *was*. Only the wiring describes what it
> is *being used as*.**
>
> The fix reads the node graph: what socket does this texture feed? Filename is a
> last resort when nothing conclusive is wired.
>
> Bonus lesson: boundary-anchor your regex. Unanchored, `ao` matches half the words
> in English and `orm` matches "platform", "normal" and "transform".

---

# Part 7 — File formats
🟡 **Working knowledge**

## The formats that matter

| Format | Role | Materials | Hierarchy | Web |
|---|---|---|---|---|
| **`.max`** | 3ds Max native | Full node graphs | Full | ❌ needs a licence to open |
| **FBX** | Interchange | **Lambert/Phong only** | Good | ❌ |
| **OBJ** | Ancient fallback | Almost none | **None** | ❌ |
| **glTF / GLB** | Web delivery | PBR metallic-roughness | Full | ✅ |
| **USD/USDZ** | Emerging | MaterialX | Full | Partial |
| **`.blend`** | Blender native | Everything | Full | ❌ |

## FBX and its central limitation

Autodesk's interchange format. Carries geometry, hierarchy and animation well.

**It cannot carry node-graph materials at all.** FBX predates node-based shading as
a concept; it has fixed slots inherited from 1990s Lambert/Phong shading. VRayMtl,
CoronaPhysicalMtl, even Blender's own Principled BSDF — all flattened or dropped on
export.

This is not a bug or a missing feature. It is architectural, and it is why this
product extracts textures separately and reattaches them by naming convention
rather than trusting FBX to carry materials.

## glTF and GLB

Khronos Group's open standard, purpose-built for web and real-time. Nicknamed "the
JPEG of 3D".

- **glTF** — JSON plus separate binary and image files
- **GLB** — everything packed into one binary file

Internal structure worth knowing:

| Term | Meaning |
|---|---|
| **Scene** | The root; lists top-level nodes |
| **Node** | A transform in the hierarchy; may reference a mesh |
| **Mesh** | A collection of primitives |
| **Primitive** | One chunk of geometry with one material |
| **Accessor** | A typed view onto raw data: positions, normals, UVs, indices |
| **Buffer view** | A slice of the binary blob |

> ### 🟡 Accessors are mesh-local
>
> An accessor's `min`/`max` give the bounds of the raw vertex data — **before** node
> transforms. To get real-world size you must walk the node hierarchy and apply
> every transform down the chain.
>
> This project learned that the hard way; see [Part 2](#units).

## Compression: two completely different things

This distinction is important and often confused.

**Draco** — compresses **geometry**. Quantizes vertex positions to a fixed grid
(14 bits gives 16,384 steps per axis, imperceptible at building scale), then
entropy-codes the connectivity. Lossy but visually free.

**KTX2 / Basis Universal** — compresses **textures**, and this is the crucial part:
it stays compressed **in GPU memory**.

A JPEG is small on disk but **decompresses to raw RGBA on the GPU**. A 1024×1024
JPEG might be 200 KB as a file and **4 MB in VRAM**. KTX2 uses a GPU-native format
that the hardware reads directly while still compressed — often 4–8× less VRAM.

> ### 🔴 Why this project's file is too big
>
> Measured: **textures are 82% of the exported GLB** (14.56 MB of 17.78 MB).
>
> Draco saved only **11%**, because it compresses geometry and geometry was 18% of
> the bytes. On a synthetic test model with *no* textures, the same setting achieved
> 4.2× — a number that does not survive contact with a real asset.
>
> KTX2 is not implemented, which is the single largest remaining gap. Without it,
> 20 materials of uncompressed 1024² RGBA is roughly 400 MB of GPU memory, and
> mid-range phones run out.

## Why not USD?

Better on paper — it can carry real node graphs via MaterialX. But Blender's USD
importer drops MaterialX materials, the 3ds Max exporter needs a licensed 2024+
install, and USDZ can't contain MaterialX at all. Not viable yet.

---

# Part 8 — Geometry reduction
🔴 **Core**

## Why reduce at all

A phone must download the model over mobile data, fit it in limited memory, and
redraw it 30–60 times a second. Source arch-viz scenes routinely have 500,000 to
2,000,000 triangles. The target here is 80,000–120,000.

The perceptual justification: at typical phone viewing distance, triangles beyond
roughly 50K cover **less than one pixel each**. The GPU dutifully processes them
and you see nothing. Meanwhile users notice **texture** sharpness immediately. So:
be aggressive with geometry, conservative with textures.

## Decimation

**Decimation** — deliberately removing triangles while trying to preserve
appearance.

**Ratio** is the fraction **kept**, which is a constant source of confusion. `0.3`
means "keep 30%, remove 70%".

## Collapse (Quadric Error Metrics)

The standard algorithm, from Garland & Heckbert (1997):

1. For each vertex, build a small matrix (a **quadric**) summarising the planes of
   all faces touching it.
2. For each edge, compute the **error** merging its two endpoints would cause — how
   far the surface would move.
3. Repeatedly collapse the cheapest edge.
4. Stop at the target count.

The result is exactly what architecture wants: **flat regions collapse
aggressively, curved detail survives.** Merging two vertices in the middle of a
flat wall moves the surface by zero, so it costs nothing and goes first.

**Collapse requires shared edges** — which is why [welding](#welding) must happen
first.

## Planar dissolve (lossless)

A different operation entirely. Merges faces that are within a few degrees of
**coplanar** into single larger faces. **No vertex moves.**

For a flat surface subdivided into hundreds of triangles, this removes edges that
describe no shape. Geometrically lossless.

> ### 🔴 Measured here
>
> | | Triangles |
> |---|---|
> | After weld | 191,310 |
> | After planar dissolve (5°) | **107,956** |
> | One object alone | 115,872 → 37,536 |
>
> **44% of the scene was redundant coplanar subdivision.** This is repair, not
> reduction — which is why this project runs it by default while keeping lossy
> collapse opt-in.

## Thin features and why they break

**Thin features** — railings, balusters, mullions, window frames, trim.

Collapse destroys them. A baluster 37 mm across has almost no internal geometry to
spare; collapsing its edges makes it vanish or twist into garbage. Yet they are
often the *bulk* of the triangle count — one railing object here is 115,872 of
191,310 triangles.

Detect them by **physical thickness**, not triangle density: measure each island's
narrowest dimension. A baluster is 37 mm, a mullion 20 mm, a facade panel 400 mm.

> ### 🟡 The target is sometimes unreachable
>
> Requesting 30% kept, this project achieved **41%**. **83% of the scene is thin
> geometry** that shatters under collapse.
>
> The pipeline reports the shortfall and its reason rather than destroying the
> model to hit a number. Getting further needs **element removal or LODs, not
> simplification.**

## LOD, instancing, billboards

**LOD** (level of detail) — multiple versions at different densities; swap by
distance. Standard in games, not yet here.

**Instancing** — draw one mesh many times with different transforms. 200 identical
balusters become one mesh plus 200 transforms. Big potential win for architecture.

**Billboard / impostor** — replace distant complex geometry with a flat image that
turns to face the camera. Standard for trees.

## meshoptimizer / gltfpack

An alternative simplifier, generally better than Blender's for thin features
because it locks boundary edges and is attribute-aware (it considers UV seams when
choosing what to collapse). Not currently installed here.

---

# Part 9 — Light, rendering and tone
🟡 **Working knowledge**

## Rasterization vs ray tracing

**Rasterization** — for each triangle, work out which pixels it covers. Extremely
fast, hardware-accelerated, approximates lighting. Every real-time renderer.

**Ray tracing / path tracing** — trace light rays and simulate bounces. Physically
accurate, slow. Offline renders.

| Renderer | Type | Speed |
|---|---|---|
| **EEVEE** (Blender) | Rasterization | Seconds |
| **Cycles** (Blender) | Path tracing | Minutes–hours |
| **V-Ray / Corona** | Path tracing | Minutes–hours |
| **three.js / WebGL** | Rasterization | Real-time |

## Samples and noise

Path tracers fire random rays. Fewer rays = noisier image. **Samples** controls how
many. Doubling samples halves noise, so quality improves slowly and expensively.
**Denoising** cleans the rest algorithmically.

## Global illumination

**GI** — light bouncing off surfaces and illuminating other surfaces. A red carpet
tints a white wall pink. Expensive but essential for realism.

Real-time renderers approximate it — the main reason web 3D looks different from an
offline render.

## HDRI and image-based lighting

**HDRI** — a high-dynamic-range 360° photograph used as both backdrop and light
source. Instead of placing lamps, you light the scene with a photograph of a real
place.

**IBL** (image-based lighting) — the technique generally.

**PMREM** — the preprocessing (blurring the environment at several roughness
levels) that makes an HDRI usable for reflections at varying roughness.

> ### 🔴 The environment *is* the material, for shiny things
>
> A rough surface reflects a blurred average of its surroundings. A smooth surface
> reflects them sharply. A **metal** surface reflects *only* its surroundings.
>
> So for anything untextured and reflective, **what you see is the environment**.
>
> This project's viewer used a 16×256 two-stop colour gradient as its environment.
> Most of the building is untextured grey at roughness 0.859 — so reflecting that
> crude gradient painted soft diagonal bands across the facade and turned every
> downward-facing surface near-black. Replacing it with a proper procedural studio
> environment fixed both.

## Tone mapping and view transform

Real light has enormous dynamic range — sunlight is thousands of times brighter
than shade. Screens have maybe 300:1. **Tone mapping** squashes one into the other.

Common curves: **Standard** (none, clips), **Filmic**, **AgX** (Blender's current
default), **ACES** (film industry).

They look visibly different. **Two systems using different tone mapping will never
match**, no matter how correct the underlying data.

> ### 🟡 In this project
>
> Blender renders cubemaps through AgX; the web viewer originally used ACES. Those
> can't agree. The viewer moved to AgX for parity.

**Exposure** — a brightness multiplier applied before tone mapping, in stops
(each stop = 2×).

## Baking

**Texture baking** — rendering a material's or lighting's appearance into a flat
image, so it can be displayed without recomputing.

Once considered essential here, now mostly avoided: most agency materials are
already flat bitmaps, and baking them re-samples good data into worse data at
10–30× the processing cost. Only genuinely procedural materials need it.

Baking **must happen after decimation** — bake first and simplifying the mesh
afterwards destroys the UVs the bake was painted into.

---

# Part 10 — Cameras, projection and depth
🟡 **Working knowledge**

## Field of view

**FOV** — how wide an angle the camera sees. Wide (90°+) exaggerates perspective;
narrow (~30°) flattens it. Photographers say "focal length" instead; they describe
the same thing.

**90° FOV is not a stylistic choice for cubemaps** — it's exactly what makes six
faces tile into a seamless cube.

## Near and far clipping planes

The camera only renders between two distances: **near** and **far**. Closer than
near, or further than far, is invisible.

They exist because the depth buffer needs a finite range.

## The depth buffer and z-fighting

**Depth buffer** (z-buffer) — for each pixel, the renderer stores how far away the
nearest surface is, so closer things correctly hide farther things.

It has finite precision, typically 24 bits. And crucially, **precision is not
distributed evenly** — perspective projection concentrates it near the camera, so
distant objects get far fewer distinct depth values.

**Z-fighting** — when two surfaces are so close together that the depth buffer
can't tell which is in front. Adjacent pixels resolve differently and you get
shimmering, flickering, stippled patterns, especially when the camera moves.

The ratio `far / near` determines how bad it gets. **Raising `near` helps far more
than lowering `far`.**

> ### 🟡 Measured here
>
> `near = 0.05, far = 1148` gives a **22,954:1** ratio — about **6 mm** of depth
> resolution at 90 m. The facade has coincident faces, so balcony slabs flickered.
>
> Fixed with a **logarithmic depth buffer** (redistributes precision across the
> range) and `near = 0.1`.

## Cubemaps and panoramas

**Cubemap** — a 360° view stored as six square images, the faces of a cube around
the viewer: `+X, −X, +Y, −Y, +Z, −Z`. Standing inside the cube and looking around
reconstructs the full view.

**Equirectangular** — the same 360° view as a single wide image, like a world map.
Simpler to store, distorted at the poles.

**Face order is a contract.** Everything producing or consuming cubemaps must agree
on it.

> ### 🟡 A wrong face orientation does not error
>
> It produces six perfectly plausible images that simply don't join up. No
> exception, no warning. The only reliable check is rendering a room with six
> known-coloured walls and asserting each face shows the right one — which this
> project does.

## Spherical coordinates

Describing camera position by angles rather than XYZ:

- **theta** — horizontal angle (compass direction)
- **phi** — vertical angle (up/down)
- **radius** — distance from the target

This is what orbit controls manipulate. Watch out: APIs commonly return radians and
accept degrees, or vice versa.

## Look-at

Building a rotation that aims a camera at a target. The subtlety is that **Blender
cameras look down their local −Z axis with +Y up** — a convention you must respect
or the camera aims backwards.

---

# Part 11 — Real-time and web delivery
🟡 **Working knowledge**

## WebGL and three.js

**WebGL** — the browser's GPU API. WebGL2 is the current baseline.

**three.js** — the standard JavaScript library wrapping it. Handles scene graphs,
materials, loaders and controls so you don't write raw GPU code.

## Draw calls

A **draw call** is one instruction from CPU to GPU: "draw this geometry with this
material". Each has overhead, and **the count often matters more than the triangle
total**. 100,000 triangles in 10 draw calls beats 10,000 triangles in 500.

Merging objects that share a material is a standard optimization.

## GPU memory

The real constraint on phones. A mid-range device might have 1–2 GB available for
textures.

The trap: **file size and GPU size are different numbers.** JPEG and PNG decompress
to raw RGBA in VRAM. A 1024×1024 texture is 4 MB in memory regardless of how small
the file was. KTX2 stays compressed on the GPU — that's its whole point.

## Mipmaps

Pre-computed smaller copies of a texture (½, ¼, ⅛…). The GPU picks the size
matching how large the surface appears on screen.

Without mipmaps, distant textured surfaces shimmer horribly as pixels get sampled
almost at random. Costs 33% more memory and is essentially always worth it.

## Frame rate

**FPS** (frames per second) — 60 is smooth, 30 acceptable, below 20 feels broken.

**Frame time** is often more useful: 60 fps = 16.7 ms per frame.

**1% low** — the average of the worst 1% of frames. This matters more than the
average, because **stutter is what people notice**. A model can average 60 fps and
feel broken if it periodically hitches to 10.

> ### 🟡 Measuring FPS honestly
>
> Many viewers render **on demand** — only redrawing when something changes. A naive
> `requestAnimationFrame` counter on an idle page then measures your monitor's
> refresh rate, not any rendering work, and confidently reports 60 fps for a model
> that stutters badly.
>
> This project samples only during interaction, plus a forced-orbit benchmark.

## Sprites and raycasting

**Sprite** — a flat image that always faces the camera. Used for markers and UI in
3D space.

**Raycast** — firing an imaginary line into the scene to find what it hits. Used
for click detection and for testing whether a marker is hidden behind a wall.

---

# Part 12 — The detection algorithms
🟡 **Working knowledge**

These come from image processing and computational geometry, not 3D modelling. They
are how this project finds floors and rooms with no metadata at all.

## Histogram

Counting how much of something falls into each of a series of ranges (bins).

**Used here:** bin all upward-facing surface area by height. Floor slabs are large
flat upward surfaces repeating at regular heights, so **storeys appear as spikes**.
That single idea is the entire floor detector.

## Rasterization (as an analysis tool)

Converting shapes into a grid of cells — drawing on graph paper. Once geometry is a
grid, image-processing algorithms apply.

**Used here:** slice the building at eye height, draw every wall crossing that
plane into a grid of 25 cm cells.

## Flood fill and connected components

**Flood fill** — the paint bucket. Start at a cell, spread to every connected
neighbour that isn't a wall.

**Connected component** — a group of cells all reachable from one another.

**Used here:** flood-fill inward from outside the building. **Anything the fill
cannot reach is enclosed — that's a room.**

## 4- vs 8-connectivity

Whether "neighbour" means only up/down/left/right (**4**) or also diagonals (**8**).

The distinction matters enormously:

- **8-connected fill** can squeeze *diagonally* between two wall cells that merely
  touch at a corner — leaking outside into the interior
- **4-connected fill** cannot

## Dilation

Thickening shapes by one cell in every direction. A morphological operation from
image processing.

> ### 🟡 One line doing a lot of work
>
> Real buildings have **coincident facade faces** — two surfaces at the same place —
> which leave hairline gaps when rasterized. Without dilation, the exterior flood
> fill escapes through one of those gaps and swallows the entire interior, reporting
> zero rooms in a building full of them.
>
> Dilate **8-connected** to close gaps; fill **4-connected** so nothing sneaks
> through corners. The two choices work as a pair.

## Distance transform

For every cell, how far to the nearest wall.

**Chamfer 3-4** is a fast approximation: cost 3 for a straight step, 4 for a
diagonal (4/3 ≈ 1.33, close to √2 ≈ 1.41). Two sweeps and you're done. Within ~2%
of true straight-line distance.

Simple breadth-first search would give **Manhattan** (city-block) distance, which
biases results toward axis-aligned corridors.

## Pole of inaccessibility

The point inside a shape **furthest from any edge**. Geographically, the point in
Antarctica furthest from any ocean.

> ### 🟡 Why not just use the centroid?
>
> The **centroid** is the average position — the balance point. For an L-shaped
> room, the centroid falls in the notch, **outside the room, inside a wall**. Put a
> camera there and it renders the inside of a brick.
>
> The pole is always inside the shape, and is also the most open place to stand —
> exactly what a camera wants.

## Intersection over Union

**IoU** — a similarity score for two shapes: `overlap ÷ combined area`. 1.0 =
identical, 0 = no overlap.

**Used here:** deciding two storeys share a floor plan. Real geometry has
floating-point noise, so identical-looking floors rarely match exactly — this
project's twelve storeys matched at **IoU 0.9937**, not 1.0. Exact comparison alone
would have found twelve distinct plans and saved nothing; IoU found one plan and
turned 120 renders into 10.

## Plane–polygon intersection

Finding where a flat plane cuts through a polygon.

The sign trick: for an edge from `a` to `b` crossing height `z`, if
`(a.z − z) × (b.z − z) > 0` both endpoints are on the same side and there's no
crossing. Only edges where that product is ≤ 0 actually cross.

> ### 🟡 Slice, don't project
>
> The alternative — flattening a wall's outline downward — paints the wall's whole
> height into the raster and **closes off doorways** the slice passes cleanly
> through, walling off rooms that are actually connected.

## Newell's method and the shoelace formula

**Newell's method** — computes a polygon's area and normal, for any number of
corners, in any orientation, even slightly non-planar. The robust general answer.

**Shoelace formula** — polygon area from corner coordinates in 2D. Named for the
criss-cross pattern of the multiplication. Used here to measure UV area.

---

# Part 13 — Blender vocabulary
🟡 **Working knowledge**

**bpy** — Blender's Python API. `import bpy` only works *inside* Blender.

**bmesh** — a lower-level mesh library for per-vertex and per-face surgery that the
normal API doesn't expose. Allocates memory outside Python's garbage collector, so
it must be freed explicitly.

**Datablock** — Blender's word for any reusable chunk of data: a mesh, a material,
an image, a scene. Datablocks can be shared between objects.

**Object vs mesh** — an *object* is a thing in the scene with a transform; the
*mesh* is its geometry datablock. Ten objects can share one mesh.

**Modifier** — a non-destructive effect stacked on an object, like a Photoshop
adjustment layer. Changes how it looks without changing the underlying data.
**Applying** a modifier bakes it in permanently.

<a name="depsgraph"></a>**Depsgraph** (dependency graph) — Blender's fully
calculated view of the scene *right now*, with all modifiers evaluated. Critical:
the raw mesh data and the depsgraph result can be completely different. Measuring
raw data on an object with a Subdivision modifier gives you the "before" while
looking at the "after".

**Operator** (`bpy.ops.*`) — a command, the same one a menu item would run. Acts on
whatever is currently **selected** and **active**, which is why scripts must set
those explicitly before calling one.

**Object mode / Edit mode** — Edit mode manipulates vertices; Object mode
manipulates whole objects. Many operators require one or the other.

**Collection** — Blender's folders for organising objects.

**`--background`** — run with no window, driven by a script.

**`--factory-startup`** — ignore the user's preferences and add-ons. Essential for
reproducibility: without it, a run on your laptop can differ from a run on a server.

**EEVEE / Cycles** — the two built-in renderers; see [Part 9](#rasterization-vs-ray-tracing).

---

# Part 14 — Arch-viz domain words
⚪ **Recognise**

Vocabulary from architecture and the visualization industry.

**Arch-viz** — architectural visualization. Producing images and animations of
buildings that don't exist yet, for marketing and design review.

**DCC** — digital content creation tool. 3ds Max, Blender, Maya, SketchUp.

**Walkthrough** — a navigable 3D experience, as opposed to a still render or video.

### Building parts

| Term | Meaning |
|---|---|
| **Storey** | One floor level of a building |
| **Slab** | The horizontal concrete floor/ceiling structure |
| **Soffit** | The underside of something overhead — a balcony or a ceiling overhang |
| **Facade** | The exterior face of a building |
| **Mullion** | A vertical bar dividing panes in a window or curtain wall |
| **Transom** | The horizontal equivalent |
| **Baluster** | One vertical post in a railing |
| **Balustrade** | The whole railing assembly |
| **Parapet** | A low wall along the edge of a roof |
| **Louvre** | Angled slats for shade or ventilation |
| **Lightwell** | A vertical shaft bringing daylight into a building's core |
| **Podium** | The wider base structure a tower sits on |
| **Massing** | The overall three-dimensional bulk of a building |
| **Site plane** | The ground/landscape mesh a building sits on |

### Indian real-estate terms

**2BHK / 3BHK** — "2 Bedroom, Hall, Kitchen". The standard way apartment sizes are
described in India. A 2BHK is roughly a two-bedroom flat.

**RERA** — the Real Estate Regulatory Authority. Some state RERAs **mandate QR
codes** on property advertisements, which is a direct tailwind for this product.

**NRI** — Non-Resident Indian. A significant buyer segment who cannot visit sites
in person and therefore depend on remote visualization.

### Production terms

**Eye height** — camera height for interior views, ~1.6–1.7 m, matching a standing
person.

**Hero object** — a visually prominent item deserving extra quality budget.

**Golden master** — a reference render used to compare pipeline output against.

---

# Part 15 — Where each concept actually bit us

The fastest way to internalise all of the above. Every one of these is a real bug
from this project.

| # | Symptom | Root concept | What was actually wrong |
|---|---|---|---|
| 1 | Building renders **black** on the web, fine in Blender | [Metallic](#metallic-is-a-hard-switch-not-a-dial) | The FBX importer set `Metallic = 1.0` on all 15 materials. Metal has no diffuse — it shows only reflections, and Blender's viewport had a studio HDRI to reflect while the web viewer did not. |
| 2 | Decimation runs, reports success, **removes almost nothing** | [Islands / welding](#islands-and-connected-components) | The mesh arrived as 115,872 disconnected triangles. Collapse needs shared edges; there were none. |
| 3 | Classification puts a railing and a wall in the same bucket | [Islands](#islands-and-connected-components) | Same cause. Triangles-per-island was `1` scene-wide before welding. |
| 4 | The **diffuse of most of the building is corrupted** | [Colour space](#the-critical-distinction) | Colour space was chosen by filename. Five of seven textures named `*_AmbientOcclusion` / `*_METALNESS` were actually wired into Base Color. |
| 5 | Textures **explode** after decimation (+21,311% UV area) | [UV tiling](#tiling) | A nearest-face UV transfer is incoherent when UVs tile 27–55×; adjacent points sampled different tiles. |
| 6 | Two meshes render **untextured** | [No UVs](#multiple-uv-channels) | Unwrapping lived inside the optimizer, so turning the optimizer off removed it. |
| 7 | **Diagonal streaks** across the facade | [Normals](#smoothing-groups-and-custom-split-normals) + [slivers](#sliver-triangles) | Every face smooth-shaded, 85% of them 68:1 slivers. Normal interpolation runs the length of each needle. |
| 8 | Balcony floors **shimmer** when orbiting | [Depth precision](#the-depth-buffer-and-z-fighting) | 22,954:1 far/near ratio left ~6 mm of depth resolution at 90 m, and the facade has coincident faces. |
| 9 | Facade shows **soft diagonal bands**; ceilings **black** | [IBL](#hdri-and-image-based-lighting) | The environment was a two-stop gradient, and untextured rough surfaces show you the environment. |
| 10 | Compression **barely helps** | [Draco vs KTX2](#compression-two-completely-different-things) | Draco compresses geometry; textures were 82% of the bytes. |
| 11 | A validation check reported the model **75× oversized** | [Local vs world space](#local-vs-world-space) | It read accessor bounds (mesh-local, inches) without applying node transforms carrying the 0.0254 scale. **The bug was in the checker.** |
| 12 | Room detection finds **zero rooms** in a building full of them | [Dilation](#dilation) | Coincident facade faces left hairline gaps; the exterior flood fill leaked in and swallowed the interior. |
| 13 | A camera renders **the inside of a wall** | [Pole of inaccessibility](#pole-of-inaccessibility) | An L-shaped room's centroid falls in the notch, outside the room. |
| 14 | Panoramas **invisible** despite rendering correctly | Scene composition | The viewer set the cubemap as background but never hid the building geometry in front of it. |

## The four transferable lessons

**1. Names are unreliable; structure is reliable.** Object names are 3ds Max
defaults. Material names are decorative. Texture names describe the download, not
the use. Every heuristic resting on a string is a last resort — see bugs 3 and 4.

**2. The obvious check is often the wrong check.** Triangle counts stay perfectly
plausible while UVs are destroyed (bug 5) and while materials are corrupted (bug 1).
Measure the property that actually matters — UV area, explicit metallic values,
world-space bounds.

**3. "Looks fine in Blender" proves less than you'd think.** Blender's viewport has
its own environment, its own tone mapping, and a Solid mode that ignores materials
entirely. Bugs 1, 7 and 9 were all visible in Blender to anyone who looked in the
right mode.

**4. Making something optional exposes what was hiding inside it.** Bug 6 existed
the moment decimation became opt-in, because a fidelity requirement had been
implemented inside an optimization stage. Audit what lives in a stage before you
allow it to be skipped.
