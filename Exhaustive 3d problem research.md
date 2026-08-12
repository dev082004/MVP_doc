# The Complete Field Guide to 3D Model Problems

A domain-by-domain, exhaustive catalogue of the concepts, failure modes, and artifacts you will encounter when authoring, converting, rendering, simulating, printing, or shipping 3D data.

> **Terminology note:** the term you may have heard as *"silver triangles"* is actually **sliver triangles** (also called *slivers*, *needles*, or *caps*). They are covered in §2.2.

---

## Table of Contents

1. [Foundations: representations and why problems arise](#1-foundations)
2. [Geometry and topology problems](#2-geometry-and-topology-problems)
3. [Transforms, coordinate systems, units and precision](#3-transforms-coordinate-systems-units-and-precision)
4. [Normals, tangents and shading](#4-normals-tangents-and-shading)
5. [UV mapping and parameterization](#5-uv-mapping-and-parameterization)
6. [Textures, materials and shaders](#6-textures-materials-and-shaders)
7. [Lighting, shadows and rendering](#7-lighting-shadows-and-rendering)
8. [Rigging, skinning and deformation](#8-rigging-skinning-and-deformation)
9. [Animation, simulation and caches](#9-animation-simulation-and-caches)
10. [Performance, memory and optimization](#10-performance-memory-and-optimization)
11. [File formats, interchange and pipeline](#11-file-formats-interchange-and-pipeline)
12. [Application-specific export and interchange behaviour](#12-application-specific-export-and-interchange-behaviour)
13. [Domain-specific problems](#13-domain-specific-problems)
14. [Cross-cutting concerns](#14-cross-cutting-concerns)
15. [Diagnostic triage table](#15-diagnostic-triage-table)
16. [Validation checklists](#16-validation-checklists)
17. [Glossary](#17-glossary)

---

## 1. Foundations

Almost every problem in 3D comes from one of six root causes. Recognizing which one you are looking at cuts debugging time enormously.

| Root cause | Description | Typical symptom |
|---|---|---|
| **Representation mismatch** | The data model you have is not the data model the consumer expects (BREP vs mesh, quads vs triangles, curve vs polyline). | Conversion loss, exploded surfaces, missing features |
| **Topological invalidity** | The mesh violates assumptions (manifoldness, orientability, closure). | Boolean fails, slicer fails, physics fails |
| **Numerical precision** | Floating point cannot represent the values involved. | Jitter, z-fighting, cracks, degenerate faces |
| **Convention mismatch** | Units, axis, handedness, color space, tangent basis, channel order. | Object 100× too big, inverted normal map, washed-out textures |
| **Sampling/discretization** | Continuous phenomena approximated by finite samples. | Aliasing, faceting, moiré, staircase artifacts, noise |
| **Budget violation** | Correct data that is simply too much for the target. | Crashes, OOM, low framerate, hours-long bakes |

### 1.1 The representations

**Polygon mesh (triangles/quads/n-gons).** Vertices + edges + faces. The lingua franca of real-time and film. Cheap, universal, but only an *approximation* of curved surfaces, and carries no notion of "solid" unless you enforce it.

**BREP (Boundary Representation).** The CAD model: exact trimmed NURBS/analytic surfaces stitched into shells with topological adjacency, plus tolerance. Exact, but heavy, kernel-specific, and fragile under translation.

**NURBS / spline surfaces.** Parametric patches with control points, knots, weights, and degree. Exact curves; problems concentrate in trimming, continuity, and patch layout.

**Subdivision surfaces (Catmull-Clark, Loop, OpenSubdiv).** A control cage plus a refinement rule. Smooth and animation-friendly; problems concentrate in cage topology (poles, n-gons, creases).

**Implicit surfaces / SDFs / volumes / voxels.** Function or grid defined; naturally watertight and boolean-friendly; problems are resolution, memory, and the meshing step (marching cubes).

**Point clouds.** Unstructured samples from scanning/LiDAR. No connectivity, no surface — everything downstream must reconstruct it.

**Gaussian splats / NeRF / neural fields.** Radiance representations. Great visual fidelity, poor geometric semantics — no usable mesh, no editability, no collision.

**Curves, hair, and instanced primitives.** Strands, ribbons, particles. Break most mesh tooling assumptions.

### 1.2 Attribute domains

A recurring source of confusion: the *domain* on which data lives.

- **Per-vertex** (shared) — position, weights.
- **Per-face** — material ID, smoothing group.
- **Per-face-vertex / per-corner** (also "vertex split", "wedge") — UVs, normals, vertex color, tangents.

Formats disagree here. OBJ stores independent index streams per attribute; glTF/real-time engines require a **single fused index buffer**, which forces vertex duplication at every UV seam, hard edge, or material boundary. The result: your 5,000-vertex model imports as 8,300 vertices, and per-vertex edits made downstream silently break.

---

## 2. Geometry and Topology Problems

### 2.1 Degenerate geometry

Any primitive with zero or effectively-zero measure.

- **Zero-area faces** — three collinear or coincident vertices. Produce undefined face normals → NaN propagation into vertex normals and tangents → black shading, broken lighting, corrupt normal map bakes.
- **Zero-length edges** — two coincident vertices connected. Break edge-collapse decimation, subdivision, and beveling.
- **Duplicate faces** — two faces on the same vertices. Cause z-fighting, doubled shading, and confuse manifold checks and slicers.
- **Faces with repeated vertex indices** (`f 1 1 2`) — invalid but common in exported/generated data.
- **Degenerate UV triangles** — nonzero in 3D, zero-area in UV space. Divide-by-zero in tangent generation → NaN tangents → black or exploding normal-mapped shading.
- **NaN / Inf positions** — often from failed simulation, division by zero, or bad import. One NaN vertex can blow up the entire bounding box, killing culling, physics, and camera framing.

### 2.2 Sliver, needle and cap triangles

These are *not* degenerate — they have area — but their **aspect ratio** is terrible.

- **Sliver / needle** — one very short edge, or a triangle that is long and thin (large aspect ratio).
- **Cap** — one angle near 180°, vertex nearly on the opposite edge.

Why they matter:

- **Rasterization**: thin triangles may cover no pixel centers → dropped/flickering geometry; they also waste GPU quad-fragment throughput (the GPU shades 2×2 quads, so a thin triangle can be ~4× more expensive per useful pixel).
- **Normal/tangent interpolation** becomes numerically ill-conditioned → shading spikes and dark bands.
- **FEA/CFD**: high aspect ratio and skewness destroy the element Jacobian → solver ill-conditioning, non-convergence, wrong stresses.
- **Baking**: barycentric interpolation error → wavy or pinched normal maps.
- **Subdivision / smoothing**: slivers amplify into visible pinches.
- **Physics**: unstable collision normals, jitter, objects sliding or vibrating on contact.

Mitigation: quality-driven remeshing, edge collapse below a length threshold, angle-based cleanup, Delaunay-style flips, minimum-angle constraints in the mesher.

### 2.3 Coplanar and coincident geometry

- **Coplanar faces** (two surfaces occupying the same plane) → **z-fighting**: the depth buffer cannot resolve which is nearer, so pixels alternate frame to frame, and the flicker changes with camera distance because depth precision is non-linear.
- **Coplanar triangles within a single flat n-gon** — harmless visually, but they waste triangles and confuse planarity-based tools.
- **Coincident/overlapping shells** — two copies of the same object stacked. Doubles poly count, breaks booleans, causes shadow acne and lightmap chaos.
- **Interior/hidden faces** — geometry inside a closed object. Invisible but costs memory, breaks watertightness checks, wrecks lightmap packing and 3D print slicing.
- **Intersecting geometry** — legal for real-time (a sword through a rock), fatal for printing, CFD, and booleans.

Mitigations: depth bias/polygon offset, decal-specific rendering paths, reversed-Z depth buffers, geometry offset by a small epsilon, or removing the duplicate outright.

### 2.4 Non-manifold geometry

A manifold mesh is one where every point has a neighborhood that looks like a flat disk (or a half-disk on a boundary). Violations:

- **Non-manifold edge** — an edge shared by 3+ faces (the "T-shape" or fin).
- **Non-manifold vertex / bowtie** — two surface cones meeting at a single vertex, faces connected only through that point.
- **Internal faces** inside a closed volume.
- **Non-orientable surfaces** (Möbius-like) — no consistent inside/outside.
- **Isolated/loose vertices and edges** — no face attached.
- **Zero-thickness walls** — a single-sided surface treated as a solid.

Consequences: booleans fail or produce garbage, subdivision produces spikes, normals cannot be consistently oriented, 3D print slicers produce undefined infill, physics engines refuse the mesh, UV unwrapping algorithms crash, and remeshers produce artifacts.

### 2.5 Holes, boundaries and watertightness

- **Boundary edges** (edges with exactly one adjacent face) define holes. Some are intentional (a plane, a cloth), some are bugs (missing face, deleted geometry, failed stitch).
- **Watertight / closed / solid** — no boundary edges, consistently oriented, positive enclosed volume. Required for: 3D printing, CFD, boolean CSG, volume/mass calculations, SDF conversion, physics convex/concave collision generation.
- **Cracks from T-junctions** — a vertex lying *on* another face's edge but not topologically connected. Under rasterization and floating-point interpolation, pixel-wide seams of background show through. Common in terrain LOD stitching, CAD tessellation, and modular kit-bashed level geometry.
- **Hole-filling artifacts** — automatic fill produces flat caps, self-intersections, or non-smooth patches; curvature-aware filling is better but slower.

### 2.6 Winding order, normals orientation and sidedness

- **Winding order** (CCW vs CW when viewed from the front) determines the face normal by convention and drives **backface culling**.
- **Inconsistent winding** across a mesh → patches of the model disappear from certain angles, or shade black.
- **Flipped normals** → object looks inside-out, lighting inverted, ambient occlusion inverted, print slicers invert solid/void.
- **Mirrored geometry with negative scale** flips winding implicitly. Engines either auto-correct or render the mirror inside-out; the same asset can behave differently in two tools.
- **Double-sided vs single-sided materials** — double-sided fixes visuals but doubles fill cost, breaks shadowing, and hides genuine normal errors.

### 2.7 Polygon types

- **Triangles** — the only primitive GPUs actually draw; everything else is triangulated eventually.
- **Quads** — preferred for modeling, subdivision, and deformation. Non-planar quads are ambiguous: which diagonal is used changes the silhouette and shading, and *different tools choose different diagonals* → your model looks different after export.
- **N-gons** (5+ sides) — convenient in modeling and CAD, dangerous in deformation and subdivision. Concave or non-planar n-gons triangulate unpredictably; ear-clipping vs monotone vs Delaunay triangulation give different results. Self-intersecting n-gons triangulate into garbage.
- **Triangulation ambiguity** is one of the most common "it looked fine in Blender/Maya but broke in the engine" causes. Fix: triangulate explicitly before export.

### 2.8 Topology flow and edge layout

- **Edge loops and poles** — for deforming meshes, loops should follow muscle/crease directions. Poles (vertices with valence ≠ 4 on a quad mesh; typically 3-poles and 5-poles) cause shading pinches and subdivision artifacts if placed in high-curvature or high-deformation areas.
- **Extraordinary vertices in Catmull-Clark** produce a curvature discontinuity at the limit surface — visible as a faint dimple on reflective materials.
- **Triangles in deforming regions** — shear differently than quads under skinning; produce creasing at elbows/knees.
- **Poor pole placement** creates star-shaped shading artifacts, especially on car-body-like continuous surfaces (class-A surfacing).
- **Uneven density** — big triangles next to tiny ones cause shading discontinuities, poor lightmap packing, and bad decimation.
- **Insufficient edge loops** near creases → soft, mushy silhouettes; too many → wasted budget and shading noise.
- **Support loops and hard-surface creasing** — the classic sub-D control problem: keeping edges crisp without pinching.

### 2.9 Boolean and CSG failures

- Fail on non-manifold, self-intersecting, open, or duplicated input.
- **Coplanar face conflicts** — booleans are least stable exactly where two surfaces touch tangentially.
- **Near-tangent intersections** produce sliver strips and near-zero-area faces.
- **Tolerance mismatch** — the operation succeeds numerically but leaves microscopic gaps.
- Result topology is usually terrible (n-gons, slivers, long thin faces) and needs retopology before deformation or subdivision.
- **Exact/robust predicates** (exact arithmetic, plane-based booleans) trade speed for reliability.

### 2.10 Self-intersection

- Modeling artifact, simulation artifact (cloth passing through itself), or offset artifact (shell/thickness operations self-intersect on concave regions with radius smaller than the offset distance).
- Breaks: printing, volume computation, collision, SDF conversion, UV unwrap.
- Detection is O(n²) naively; needs spatial acceleration; robust repair is genuinely hard.

### 2.11 Retopology, decimation and remeshing

- **Decimation artifacts** — feature loss, silhouette collapse, UV distortion, boundary erosion, normal flipping on collapse, texture-attribute smearing.
- **UV-aware vs UV-ignorant decimation** — the latter destroys texture mapping.
- **Quadric error metric** performs well on smooth surfaces, poorly on hard edges without edge-weighting.
- **Voxel remeshing** guarantees manifoldness but loses sharp features and produces uniform, off-model density.
- **Auto-retopology** rarely respects edge flow requirements for animation.
- **LOD generation** — see §10.4 for popping and cross-LOD consistency issues.

### 2.12 Curved-surface (CAD/NURBS) specific geometry problems

- **Trimmed surface issues** — trim curves in parameter space that do not exactly match adjacent surfaces in 3D → gaps or overlaps at tolerance boundaries.
- **Tangency / continuity classes**: G0 (positional), G1 (tangent), G2 (curvature), G3 (curvature rate). Class-A surfacing requires G2/G3; failing it produces visible "highlight breaks" in reflections even though the geometry looks fine in shaded view.
- **Degenerate patches** — collapsed edges (a NURBS "pole", e.g. sphere caps) create parameterization singularities: texture pinching, offset failure, meshing artifacts.
- **Sliver faces and short edges in BREP** — often introduced by imports; they defeat filleting, shelling, and meshing.
- **Knot multiplicity and self-overlapping control polygons** → kinks and wobbles.
- **Tolerance and healing** — every kernel has a model tolerance (commonly ~1e-6 to 1e-3 units). Geometry authored at a different tolerance imports as "not stitched" → free-floating surfaces instead of a solid.
- **Tessellation/faceting** — converting BREP to mesh uses chord height, angle, and aspect-ratio tolerances. Too coarse → visibly faceted curves and inaccurate simulation; too fine → hundreds of millions of triangles. Faceting is also *view-independent* once baked, so a "close-up" needs a re-tessellation.
- **Feature tree / parametric failures** — rebuild errors, over- or under-defined sketches, features referencing deleted topology ("topological naming problem"), broken external references in assemblies.

---

## 3. Transforms, Coordinate Systems, Units and Precision

### 3.1 Convention mismatches

- **Up axis**: Y-up (Maya, Unity, glTF, most game engines) vs Z-up (Blender, 3ds Max, CAD, USD default in many pipelines, GIS). Wrong assumption → model lying on its side.
- **Handedness**: right-handed (OpenGL, Maya, Blender) vs left-handed (DirectX, Unity, Unreal). Converting requires flipping one axis, which flips winding order — so a naive conversion gives you a correctly-oriented but inside-out model.
- **Forward axis**: +Z vs -Z vs +X. Characters import facing backwards; vehicles drive in reverse.
- **Unit scale**: meters, centimeters, millimeters, inches, "generic units". Classic failures: a 1.8 m character imports as 1.8 cm; Unreal's default 1 uu = 1 cm vs Unity's 1 unit = 1 m → 100× errors. FBX has a unit scale factor that many exporters write incorrectly.
- **Scale affects physics and lighting**, not just size: gravity, drag, inverse-square light falloff, depth-of-field, subsurface scattering distance, fog density, cloth stiffness, and physically-based camera exposure are all scale-dependent. A 100× scale error makes lighting look "wrong" in a way that is hard to diagnose.
- **Winding + normal flip on axis conversion** — always verify after any axis conversion.

### 3.2 Transform-matrix problems

- **Non-uniform scale** — breaks the naive normal transform (normals must use the inverse-transpose), shears child objects, breaks physics colliders, and confuses skinning and IK.
- **Negative scale / mirroring** — flips winding, inverts normals, may invert tangent handedness, breaks some exporters entirely.
- **Shear** — arises from non-uniform scale in a rotated parent chain. Many formats (glTF, most engines) cannot represent shear in a TRS decomposition, so it is silently baked, approximated, or dropped.
- **Unfrozen / unapplied transforms** — the mesh looks right but has hidden rotation/scale; downstream tools that read raw vertices get wrong data.
- **Pivot / origin placement** — wrong pivot breaks rotation, scaling, snapping, instancing, and gameplay attachment. Objects far from their own origin also lose precision.
- **Non-orthonormal matrices** — accumulated floating-point drift after thousands of concatenated rotations; fix by re-orthonormalizing.
- **Deep hierarchies** — transform cost, precision loss, and hard-to-debug compound offsets.
- **Baked vs live transforms** — animation curves on a parent that gets baked out lose their motion.

### 3.3 Floating-point precision

- **32-bit float has ~7 decimal digits.** At 100,000 units from the origin, the spacing between representable values is around 0.0078 units — bigger than the detail you're modeling.
- **Symptoms**: vertex jitter, camera shake, jittering shadows, z-fighting far from origin, cracked seams between adjacent objects, animation stutter, physics instability.
- **Fixes**: floating origin / origin rebasing, double precision on CPU with float rendering relative to camera, tiled/local coordinate systems, per-object local space, world-space partitioning. Essential for open-world games, GIS, BIM, planetary and astronomical scenes.
- **Depth buffer precision** — non-linear distribution wastes precision far from the camera. A near plane of 0.01 with a far plane of 100,000 guarantees z-fighting. Fixes: raise the near plane, use reverse-Z with a float depth buffer, use logarithmic depth, or split into depth ranges.
- **Catastrophic cancellation** in nearly-degenerate computations (normals of thin triangles, ray-plane intersection at grazing angles, barycentric coordinates near an edge).
- **Half-precision (fp16)** on mobile/mediump shaders: banding, position artifacts, lost UV precision on large-tiling textures.
- **Quantized vertex formats** (16-bit positions, 10-10-10-2 normals, compressed UVs) — cracks between meshes, visible stair-stepping, seams between LODs.

---

## 4. Normals, Tangents and Shading

### 4.1 Normals

- **Face vs vertex normals** — flat vs smooth shading. Vertex normals are computed by averaging adjacent face normals, optionally area- or angle-weighted; different weighting schemes give visibly different shading, and different tools use different defaults.
- **Smoothing groups / hard edges / split normals** — the mechanism for making some edges sharp. Formats disagree: 3ds Max uses smoothing-group bitmasks, Maya uses hard/soft edge flags, glTF/engines use duplicated vertices. Round-tripping loses or scrambles this.
- **Custom / authored normals** — hand-adjusted normals (very common for foliage, hair cards, and hard-surface "fake bevels") get destroyed by any operation that recomputes normals: import settings, mirroring, decimation, triangulation.
- **Auto-smooth angle** thresholds produce different results at different mesh densities.
- **Faceting** — visible flat facets on what should be a curved surface; either insufficient geometry, a wrong smoothing angle, or lost normals.
- **Shading artifacts from n-gons and non-planar quads** — the interpolation across a non-planar polygon is undefined until triangulation.
- **Normal averaging across UV seams** — if the seam splits vertices, normals split too and a visible shading seam appears unless normals are re-averaged across the split.
- **Zero-length / NaN normals** from degenerate faces.
- **Normals not renormalized after skinning or non-uniform scale** → brightness shifts during animation.

### 4.2 Tangent space

- **Tangent basis** (tangent, bitangent, normal) is required for normal mapping. It is derived from UVs, so **a normal map is only valid with the tangent basis it was baked against.**
- **Mismatched tangent basis** between baker and renderer is the single most common cause of subtle normal-map shading errors: faceted-looking bakes, dark/light bands along seams, incorrect curvature.
- **MikkTSpace** is the de facto standard (used by glTF, Blender, Unity, Unreal, Substance) precisely to solve this. Any tool that computes tangents differently reintroduces the problem.
- **Bitangent handedness / mirrored UVs** — mirrored UV shells flip handedness; storing the sign in the w component of a 4-component tangent is the standard fix. Losing that sign inverts lighting on the mirrored half.
- **Green channel convention: OpenGL (+Y) vs DirectX (-Y)** — inverted green channel makes bumps look like dents. Extremely common when moving assets between engines or from a baker to an engine.
- **Degenerate UV triangles** → undefined tangents → NaN.
- **Tangents not recomputed after deformation** — usually fine, occasionally visible on extreme deformation.

### 4.3 Normal map baking artifacts

- **Cage / ray distance problems** — rays miss the high-poly (holes, black or background-colored patches) or hit the wrong surface (a nearby finger, the inside of a hollow).
- **Skewing / projection distortion** — rays cast along averaged normals hit the high-poly at an angle, smearing detail; fixed by a manually adjusted cage or by splitting the bake with exploded meshes / bake groups.
- **Waviness** on low-density low-polys due to normal interpolation across large triangles.
- **Seams** where UV shells split — no continuity of tangent space across the seam.
- **Padding / dilation** insufficient → background bleeds in at lower mip levels.
- **Ambient occlusion / curvature bakes** — self-occlusion from geometry that should be a separate object; contact shadows baked in that should be dynamic.
- **Object-space vs tangent-space normal maps** — object-space is deformation-incompatible; tangent-space needs a consistent basis.
- **Mixing baked and detail normals** — naive addition is wrong; use whiteout/reoriented normal blending.

### 4.4 Shading interpolation artifacts

- **Shadow terminator artifact** — with low-poly geometry and smooth normals, shadow rays self-intersect near grazing angles, producing a serrated dark band. Mitigations: shading-normal-aware shadow terminator fixes, ray offsets, more geometry.
- **Specular aliasing / shimmering** — high-frequency normals plus a tight specular lobe alias badly under motion. Fix: normal-map/roughness mip filtering (Toksvig, LEAN/CLEAN mapping, specular anti-aliasing), roughness clamping.
- **Interpolation of normals across large triangles** produces "candle wax" curvature errors.
- **Curvature discontinuity at extraordinary vertices** on subdivision surfaces.

---

## 5. UV Mapping and Parameterization

### 5.1 Core UV problems

- **Stretching and compression** — texel density varies across the surface; checkers show elongated squares. Causes blurring in some regions and wasted resolution in others.
- **Non-uniform texel density across assets** — a wall at 512 px/m next to a crate at 2048 px/m looks wrong regardless of texture quality. Establish and enforce a project-wide texel-density standard.
- **Overlapping UVs** — deliberate (mirroring, tiling, reusing shells to save memory) or accidental. Fatal for: lightmap baking, ID/AO/normal baking, per-pixel unique data, texture painting.
- **UVs outside 0–1** — fine for tiling, meaningful for UDIM, catastrophic if the engine expects a single tile with clamp mode.
- **Flipped / mirrored shells** — invert tangent handedness and mirror any directional detail (text, logos, anisotropy).
- **Seams** — visible discontinuities in color, lighting, or normals. Minimize count, hide in occluded/high-contrast areas, and always add **padding/gutter** (typically 4–16 px at the target resolution, scaled per mip level).
- **Mip bleeding** — as mips shrink, neighboring shells blend into each other; padding must survive to the smallest used mip.
- **Bilinear filtering at the 0–1 border** with wrap mode samples the opposite edge → a bright line at the edge of a tiled texture.
- **Packing efficiency** — wasted UV space is wasted memory; automatic packers trade time against density.
- **Degenerate / zero-area UV faces** — NaN tangents, black bakes, unwrap failures.
- **UV distortion around poles** on spherical/cylindrical projections — pinching at the caps.
- **Lost UV sets** — many formats support multiple UV channels (UV0 diffuse, UV1 lightmap, UV2 detail/decals); exporters and engines commonly drop or reorder them.
- **UV precision** — 16-bit or half-float UVs on large tiling values cause visible stepping.
- **Vertex splitting cost** — every UV seam duplicates vertices in the render buffer, so seam count directly increases memory and skinning cost.

### 5.2 Lightmap UV specifics

- Must be **non-overlapping**, fully inside 0–1, with padding relative to lightmap resolution, and with shells aligned to reduce seam count.
- **Light leaking / bleeding** between shells that are packed too close for the chosen lightmap resolution.
- **Lightmap seams** on continuous surfaces — mitigated by seam-stitching passes in the baker.
- **Charts split across different scales** produce inconsistent bake quality on the same object.

### 5.3 Alternative parameterizations

- **Triplanar / world-space projection** — no UVs needed, no seams, but texture swims under animation, blends can look mushy, and it costs 3 texture samples per map.
- **UDIM / tiled workflows** — high fidelity for film, but many real-time engines and web formats do not support it; conversion requires atlas re-packing and re-baking.
- **Ptex** — per-face textures, no UVs; excellent for film, essentially unsupported in real time.
- **Vertex color / vertex-attribute shading** — resolution is limited by mesh density.
- **Texture atlases** — save draw calls but break tiling and complicate mipping and streaming.

---

## 6. Textures, Materials and Shaders

### 6.1 Color management

- **sRGB vs linear** — color/albedo/emissive textures are sRGB-encoded; data maps (normal, roughness, metallic, AO, height, masks) are **linear** and must never be gamma-decoded. Tagging a roughness map as sRGB is a very common, very subtle error.
- **Double gamma correction** — washed-out or overly dark results, usually from an engine that decodes an already-decoded texture.
- **Working color space and rendering space** — ACEScg, Linear sRGB, Rec.709/2020, DCI-P3. Assets authored in one and rendered in another shift hue and saturation.
- **OCIO / ACES config mismatch** across a team → each artist sees different colors.
- **Tone mapping and view transform** (Filmic, ACES, AgX, Reinhard) — the same render "looks different" purely from the display transform; artists compensate by breaking the underlying material values.
- **Display-referred vs scene-referred** confusion — clipping highlights, losing HDR range.
- **HDR / EDR display pipeline** — values above 1.0, PQ vs HLG, headroom mismatch.

### 6.2 Texture data problems

- **Resolution vs memory** — a 4K RGBA8 texture with mips is ~22 MB; a hero asset with 6 such maps is over 130 MB.
- **Non-power-of-two textures** — no or partial mip support on some hardware/APIs, compression restrictions.
- **Missing mipmaps** → severe aliasing and shimmering in the distance and huge cache-miss cost.
- **Aliasing and moiré** on high-frequency patterns (grids, fabrics, brick).
- **Compression artifacts** — BC1/DXT1 block artifacts and green-shift on gradients; BC5 for normal maps (2 channels, reconstruct Z); BC7 for high quality; ASTC on mobile; ETC2 legacy Android. **Compressing normal maps as BC1 causes blotchy banding.**
- **Compressing data maps as color-compressed formats** damages precision in roughness/metallic/masks.
- **Banding** in low-bit-depth gradients (skies, soft lighting) — fix with dithering or higher bit depth.
- **Wrong channel packing conventions** — ORM (Occlusion/Roughness/Metallic) vs RMA vs MRA vs custom; a swapped channel silently ruins the material.
- **Roughness vs glossiness/smoothness inversion** — a mirror becomes a chalk surface.
- **Bit depth for displacement/height** — 8-bit height maps cause visible stair-stepping; use 16-bit.
- **Alpha semantics** — straight vs premultiplied alpha; alpha as opacity vs alpha as mask vs alpha as smoothness (legacy conventions).
- **Missing/broken texture paths** — absolute paths from another machine, case-sensitivity differences (Windows vs Linux servers), spaces and non-ASCII in filenames, embedded vs external textures.
- **Texture streaming** — pop-in, budget overruns, wrong mip requested by an off-screen object.
- **Tiling repetition artifacts** — visible grid over large surfaces; fixed by stochastic/hex-tiling, detail-map breakups, or macro-variation.
- **Seams in cube maps and equirectangular HDRIs** — pole pinching, visible seam line, low resolution at horizon.

### 6.3 Material model problems

- **PBR workflow mismatch** — Metal/Roughness vs Specular/Glossiness. Converting between them is lossy and produces wrong edge behavior for metals.
- **Physically implausible albedo** — real-world albedo lives roughly in the 30–240 sRGB range; pure black or pure white albedo breaks GI energy and bounce lighting.
- **Baked lighting in albedo** — shadows/AO painted into the base color double-darken under real lighting and destroy relighting.
- **Energy conservation violations** — hand-authored specular or emission exceeding incoming light causes GI to blow up or never converge.
- **Metalness misuse** — partial metalness values (0.3–0.7) are physically meaningless except at transition pixels; they cause dull, plasticky metals.
- **IOR / specular level confusion** between renderers.
- **Transparency** — alpha blending requires per-object sorting (which is per-triangle wrong), causes overdraw, breaks with intersecting transparent objects, and does not write depth. Alpha testing causes hard edges and breaks with MSAA (mitigate with alpha-to-coverage). Order-independent transparency is expensive.
- **Refraction and thin-film / glass** — screen-space refraction misses off-screen geometry; thin vs thick glass models differ per renderer.
- **Subsurface scattering** — scale-dependent radius, differing models (diffusion profile vs random walk) across renderers; wrong scale gives waxy or opaque skin.
- **Anisotropy** — direction driven by tangents, so it inherits every UV/tangent problem.
- **Clearcoat, sheen, iridescence, and other layers** — support varies wildly, silently dropped in export.
- **Emissive** — units mismatch (nits vs arbitrary multipliers), no actual light contribution unless GI/emissive lights are enabled.
- **Displacement vs bump vs normal vs parallax** — displacement changes silhouette and needs tessellation and watertight adjacency (or cracks appear at patch boundaries); parallax occlusion breaks at silhouettes and grazing angles; bump is cheapest and flattest.
- **Tessellation cracks** — adjacent patches subdividing at different levels; solved by edge-factor matching / crack-free displacement.
- **Node graph portability** — procedural material graphs are almost never portable between DCCs and engines; you must bake to textures at some cost in flexibility.
- **Shader permutation explosion** — thousands of variants, long compile times, hitching on first use; mitigated by PSO precaching.
- **Shader compilation stutter** and platform-specific shader compiler bugs.

---

## 7. Lighting, Shadows and Rendering

### 7.1 Shadow problems

- **Shadow acne** — self-shadowing from depth-map quantization; fixed with depth bias, normal-offset bias, or front/back-face culling in the shadow pass.
- **Peter-panning** — too much bias detaches the shadow from the object.
- **Shadow map aliasing / blocky edges** — insufficient resolution; mitigated with cascaded shadow maps, PCF, PCSS, VSM/ESM.
- **Cascade transitions** — visible seams and popping between cascades.
- **Light leaking** — light passing through thin/single-sided walls in lightmaps, probes, and screen-space GI. Real geometric thickness is the reliable fix.
- **Shadow flickering** — from unstabilized cascades that shift with camera motion.
- **Missing shadows from double-sided or non-shadow-casting geometry.**
- **Contact shadows / SSAO artifacts** — halos, self-occlusion darkening, view-dependent flicker.

### 7.2 Global illumination and baking

- **Bake times** — hours to days on large scenes; iteration bottleneck.
- **Lightmap seams and leaks** (see §5.2).
- **Lightmap resolution vs memory** budget.
- **Light probe placement** — bad placement causes objects to be lit by the wrong room; probe interpolation over walls.
- **Dynamic vs static mismatch** — a baked scene where an object moves reveals its baked shadow.
- **Path tracing noise / fireflies** — very bright, sparsely-sampled paths; mitigate with clamping (biases results), better sampling (MIS, NEE), or denoising.
- **Denoiser artifacts** — blotching, loss of detail, temporal smearing, flickering on animation.
- **Caustics** — extremely slow to converge; often faked.
- **Non-deterministic renders** — different sampling per frame causes temporal flicker in image sequences; fix with fixed seeds/blue-noise sequences.
- **Ray-tracing acceleration structure cost** — BLAS/TLAS build time and memory for high-poly or heavily-animated scenes.
- **Self-intersection / shadow terminator** in ray tracing (see §4.4).

### 7.3 Camera, depth and visibility

- **Z-fighting** (see §2.3 and §3.3).
- **Near/far plane clipping** — geometry disappearing at the camera or in the distance.
- **Backface culling errors** with flipped winding.
- **Frustum, occlusion and distance culling errors** — objects popping in/out, incorrect bounding volumes (especially after skinning or vertex-shader displacement, which the CPU-side bounds do not know about).
- **Incorrect bounding boxes** cause both wrong culling and wrong shadow-caster selection.
- **Lens distortion and camera matching** — VFX integration requires undistort/redistort; mismatched sensor size or focal length breaks the track.

### 7.4 Anti-aliasing and temporal artifacts

- **Geometric aliasing** on thin geometry (wires, foliage, railings).
- **MSAA** doesn't fix shader aliasing and is expensive with deferred rendering.
- **TAA** — ghosting, smearing on fast motion, blurring, disocclusion artifacts, poor behavior with alpha-tested foliage.
- **Upscalers (DLSS/FSR/XeSS/TSR)** — need correct motion vectors; missing or wrong motion vectors on vertex-animated, skinned, or shader-displaced geometry produce heavy smearing.
- **Specular and normal-map aliasing** (see §4.4).
- **Flickering from thin triangles** dropping below pixel coverage.

### 7.5 Lighting setup problems

- **Unit mismatch** — lumens vs candela vs watts vs arbitrary intensity; importing a light rig between renderers gives absurd brightness.
- **Inverse-square falloff and scene scale** — light range behaves entirely differently at the wrong scale.
- **IES profiles** unsupported or mis-oriented.
- **Too many overlapping dynamic lights** — forward rendering cost explosion; clustered/deferred mitigations.
- **Exposure and auto-exposure** hunting, flickering when a bright object enters frame.
- **HDRI environment issues** — low dynamic range "HDR" files (clipped sun), wrong rotation, wrong intensity, visible resolution in reflections, baked-in objects.

---

## 8. Rigging, Skinning and Deformation

### 8.1 Skeleton and bind problems

- **Bind pose / rest pose mismatch** — the inverse bind matrices don't match the skeleton the mesh was bound to → the character explodes into a spiky mess on the first frame. The classic "shattered character" import bug.
- **T-pose vs A-pose** conventions differ; retargeting between them introduces shoulder errors.
- **Joint orientation** — inconsistent local axes make animation curves unintuitive and break mirroring and retargeting.
- **Gimbal lock** — Euler rotations lose a degree of freedom when two axes align; causes sudden flips. Mitigations: proper rotation order per joint, quaternions, or additional gimbal-avoidance controls.
- **Quaternion interpolation problems** — double cover (q and −q are the same rotation) causes "long way round" flips unless the shortest path is enforced; slerp vs nlerp differences; quaternion compression artifacts.
- **Scale in joint hierarchies** — "segment scale compensate" exists in Maya, doesn't map cleanly to other tools; non-uniform joint scale is unsupported or broken in many engines.
- **Bone count limits** — engines cap bones per skeletal mesh/draw call; exceeding it forces splitting.
- **Naming mismatch** — retargeting and tooling are name-driven; a single renamed joint breaks the chain.
- **Extra/unsupported nodes** — helper joints, twist joints, constraints, and IK handles may not export.

### 8.2 Skinning artifacts

- **Candy-wrapper twist** — linear blend skinning collapses volume to zero when a joint twists ~180°. Fixed with twist/roll joints, dual-quaternion skinning, or corrective shapes.
- **Volume loss at bends** — elbows and knees pinch. Fixed with helper joints, corrective blendshapes, or delta mush.
- **Joint collapse / candy-wrap at shoulders and wrists** — the hardest area in character rigging.
- **Bulging from dual-quaternion skinning** — DQS fixes twist but introduces bulging on bends.
- **Influence limits** — engines commonly allow 4 (sometimes 8 or 12) bone influences per vertex; exceeding it silently drops the smallest weights, changing deformation.
- **Non-normalized weights** — weights not summing to 1.0 cause vertices to shrink toward or explode away from the origin.
- **Stray weights** — a single vertex weighted to a distant bone leaves a long spike during animation.
- **Weights lost/scrambled by topology edits** after skinning.
- **Symmetry breaking** — mirrored weights that aren't quite mirrored.
- **Normals not correctly skinned** — normals must be transformed by the same blended matrices (inverse-transpose); skipping this makes lighting swim.
- **Cloth/hair penetration through the body** during deformation.

### 8.3 Blendshapes / morph targets

- **Vertex-order dependency** — blendshapes are per-vertex deltas keyed by index. Any operation that reorders, adds, or removes vertices (triangulation, merging, decimation, re-export) invalidates them.
- **Topology change** breaks all shapes.
- **Combinatorial explosion** — corrective shapes for combinations of expressions grow factorially; managed with combination/in-between systems.
- **Memory cost** — full-mesh deltas per shape; mitigated by sparse deltas.
- **Interpolation is linear**, so large rotational shapes (jaw, arm) look wrong; needs joints or in-betweens.
- **Normals and tangents on blendshapes** — often not blended, causing lighting to lag the shape.
- **Export limits** — some formats and engines cap shape count or drop names/ordering.

### 8.4 Retargeting

- **Proportion differences** — different limb lengths cause foot sliding, floating, or self-intersection.
- **Hip/root height mismatch** — character sinks into or hovers above the floor.
- **Foot sliding** — root motion speed not matching stride length.
- **IK vs FK mismatch** — baked FK curves ignore IK targets; contact points drift.
- **Different rotation orders and joint orientations** between source and target.
- **Root motion** — whether motion lives in the root or the hips, whether it is baked, and whether the engine consumes it. Mismatch causes characters to run in place or slide across the level.
- **Mocap data problems** — marker swaps, occlusion gaps, jitter/noise, foot skating, floor penetration, scale calibration errors, and finger/facial data of poor quality.

---

## 9. Animation, Simulation and Caches

### 9.1 Animation data

- **Frame rate mismatch** — 24 vs 25 vs 30 vs 60 fps; import at the wrong rate makes animation too fast/slow or introduces resampling judder.
- **Non-integer frames / sub-frame samples** dropped by exporters.
- **Curve interpolation loss** — Bezier/TCB/spline tangents are commonly baked to linear or per-frame samples on export, inflating file size and losing editability.
- **Baking necessity** — constraints, IK, expressions, drivers, and procedural rigs generally must be baked to joint transforms for engine consumption; anything not baked silently disappears.
- **Animation compression artifacts** — keyframe reduction and quantization cause jitter, foot sliding, and finger pops. Error thresholds must be per-bone (a 1 mm error at the wrist matters more than at the hip... and vice versa for accumulated hierarchy error).
- **Looping discontinuities** — first and last frames not matching; velocity mismatch even when positions match.
- **Additive animation reference-pose mismatch.**
- **Blending artifacts** — linear blending of rotations across a large angle; foot/hand IK not preserved during blends.
- **Time units** — ticks, seconds, frames; FBX time modes are a perennial source of error.
- **Animation and scale** — animation authored at one unit scale applied at another.

### 9.2 Physics and simulation

- **Collision geometry vs visual geometry** — engines need convex hulls or simplified primitives; concave collision is expensive or restricted to static objects. Using the render mesh for collision is a classic performance disaster.
- **Tunneling** — fast objects passing through thin colliders; fixed with continuous collision detection or thicker colliders.
- **Instability / explosion** — from timestep too large, stiffness too high, degenerate geometry, bad mass ratios, or interpenetrating initial states.
- **Interpenetration at frame 0** — the most common cause of "the sim blew up".
- **Self-collision cost** in cloth and hair.
- **Non-determinism** — floating point across platforms and threading order; a problem for lockstep multiplayer and reproducible sims.
- **Substep and solver-iteration tuning** trade cost against stability.
- **Inertia tensors and center of mass** wrong for a mesh with non-uniform density assumptions.
- **Scale-dependent behavior** — gravity, damping, and stiffness all depend on unit scale.
- **Collision margins** producing visible floating or sinking.
- **Cloth/hair**: pinching, stretching (needs strain limiting), volume loss, layering order, wind response, LOD popping between sim and animation.
- **Destruction/fracture**: interior faces need UVs and materials, fragment count explosion, non-watertight fragments.
- **Fluid/smoke**: voxel resolution vs memory, dissipation, boundary conditions, mesh conversion artifacts.

### 9.3 Caches and point-cached geometry

- **Alembic / USD / VDB caches** — enormous file sizes for per-frame vertex data; disk streaming becomes the bottleneck.
- **Topology-varying caches** break most consumers.
- **Missing velocity/motion-vector data** → no motion blur, broken temporal upscaling.
- **Sample-rate and time-offset mismatch** between caches and the shot.
- **Attribute loss** — UVs, colors, and custom attributes not written or not read.

---

## 10. Performance, Memory and Optimization

### 10.1 Geometry cost

- **Polygon budget** — but triangle *count* alone is misleading. What actually costs: small triangles (quad overshading), overdraw, vertex attribute size, and draw call count.
- **Vertex count inflation** from UV seams, hard edges, and material splits (see §1.2). A 10k-vertex model can become 18k in the GPU buffer.
- **16-bit index buffers** cap at 65,536 vertices per mesh; exceeding it forces 32-bit indices (double memory) or a mesh split.
- **Vertex cache locality** — unoptimized index order causes repeated vertex shading; fixed with vertex-cache-optimizing reorder (Forsyth/Tipsify) and overdraw optimization.
- **Attribute bloat** — unused UV channels, full-float normals/tangents, unused vertex colors.
- **Skinned mesh cost** scales with vertex count × influences × passes (including shadow passes).

### 10.2 Draw calls and batching

- **Too many draw calls / material slots** — each material on a mesh is a separate draw call.
- **Batching broken by** unique material instances, non-uniform scale, dynamic objects, different lightmaps, or per-object parameters.
- **GPU instancing** requires identical mesh+material; small variations defeat it.
- **State changes** (shader, texture, blend mode) cost more than triangles.

### 10.3 Memory

- **Texture memory** usually dwarfs geometry memory.
- **Duplicate assets** — the same texture imported five times under different names.
- **Uncompressed or wrongly compressed textures.**
- **Mobile/VR/web constraints** — hard memory ceilings, thermal throttling, tile-based deferred renderers punishing overdraw and mid-frame render-target changes.
- **Load times and hitching** from synchronous asset loads and shader compilation.

### 10.4 LOD and streaming

- **Popping** at LOD transitions; mitigated with dithered cross-fade or smooth geometric transition.
- **Silhouette collapse** in aggressive LODs.
- **UV/normal degradation** across LODs — shading changes visibly.
- **LOD/collision/shadow mismatch** — shadows cast by LOD0 while LOD3 renders.
- **Skinned LODs** need weight remapping and can deform differently.
- **Cracks between LODs of adjacent tiles** (terrain) — solved with skirts, stitching, or geomorphing.
- **Impostors/billboards** — parallax errors, lighting mismatch, silhouette flatness.
- **Virtualized geometry (Nanite-style)** — solves triangle density but has its own constraints: poor with translucency, deforming meshes, very small instanced foliage, and it changes how LOD authoring works.
- **Streaming pop-in** and budget overrun.

---

## 11. File Formats, Interchange and Pipeline

### 11.1 Format-specific pitfalls

| Format | Strengths | Problems |
|---|---|---|
| **OBJ** | Universal, simple, human-readable | No animation, no rig, no scene graph; materials via fragile MTL; separate index streams; no units; no color space; huge files; ambiguous n-gon handling |
| **STL** | Universal for printing | Triangle soup — no shared vertices, no units, no color (mostly), no normals worth trusting, no metadata; every consumer must weld and repair |
| **PLY** | Simple, supports arbitrary attributes | No scene graph, no materials, weak standardization |
| **FBX** | De facto DCC interchange, rigs + anim | Proprietary, version fragmentation (2011/2013/2016/2020...), ASCII vs binary, unit and axis handling bugs, inconsistent smoothing-group export, opaque failures, no formal spec |
| **glTF / GLB** | Modern, PBR-native, web/real-time standard, well-specified | Deliberately limited: no NURBS, no procedural materials, limited to metal/rough (+ extensions), no scene-level DCC features; extension support varies; triangles only |
| **USD / USDZ** | Composition, layering, huge scenes, industry-adopted | Complexity, steep learning curve, composition-arc surprises, variable renderer support, versioning of layers, USDZ constraints |
| **Alembic** | Robust baked caches | No rigs (baked only), enormous files |
| **COLLADA (DAE)** | Open, XML | Inconsistent implementations, verbose, largely superseded |
| **STEP / IGES** | CAD interchange (BREP) | Translation loss, healing needed, no feature tree, no parametrics, tolerance mismatches; IGES especially prone to unstitched surfaces |
| **JT / 3D XML / Parasolid / ACIS** | Kernel/vendor native | Licensing, version compatibility |
| **3MF** | Modern printing format (units, color, materials, manifold-aware) | Less universal than STL |
| **IFC** | BIM semantics | Huge, semantically complex, inconsistent exporters |
| **DCC natives (.blend/.ma/.max/.c4d)** | Full fidelity | Not portable, version-locked, not diffable |

### 11.2 Conversion loss

Every export/import is lossy. Commonly lost or corrupted:

- Custom/split normals and smoothing groups
- Secondary UV sets and vertex color channels
- Material graphs, procedural nodes, layered shaders
- Constraints, drivers, expressions, IK setups
- Instancing (becomes duplicated geometry, exploding file size)
- Object/component naming and hierarchy
- Units and axis orientation
- Curve/NURBS data (becomes polylines/meshes)
- Custom attributes and metadata
- Blendshape names and ordering
- Animation curve tangents
- Layer/collection/visibility state

**Round-tripping is not safe.** A→B→A is rarely identity. Keep a source of truth in the native format and treat exports as build artifacts. For the application-by-application detail of *how* each tool writes and reads these formats, see **§12**.

### 11.3 Naming, organization and hygiene

- **Duplicate names** — breaks referencing, retargeting, and material assignment.
- **Special characters, spaces, non-ASCII, case sensitivity** — break on Linux build servers and in some engines.
- **Path length limits** on Windows.
- **Absolute vs relative texture paths.**
- **Unnamed/default names** (`Cube.001`, `Material.003`) make automation impossible.
- **No naming convention** for LODs, collision meshes, sockets — most engines rely on prefixes/suffixes (`UCX_`, `_LOD0`, `SM_`, `SK_`).

### 11.4 Version control and collaboration

- **Binary files don't merge** — no diffing, no conflict resolution; requires locking workflows.
- **Repository bloat** — every save of a 500 MB scene is a new blob; needs LFS, Perforce, or asset-server strategies.
- **Broken references** when files move.
- **Non-deterministic exports** — the same source producing different bytes each export defeats caching and diffing.
- **Dependency tracking** — which textures does this material actually use?
- **Asset validation gates** — the only reliable defense; see §16.

### 11.5 Security and legal

- **Malicious files** — some formats can carry scripts (embedded Python in .blend, MEL/Python in Maya scenes, macros in CAD files). Never open untrusted scenes with auto-run enabled.
- **Decompression bombs / malformed files** crashing importers.
- **Licensing** — asset store licenses, model provenance, scanned real-world objects (trademarks, likeness), scanned people (biometric/privacy law), and AI-generated geometry provenance.
- **Export control and confidentiality** for engineering CAD data.

---

## 12. Application-Specific Export and Interchange Behaviour

§11 covered what formats can and cannot carry. This section covers the harder problem: **two applications can both write "valid FBX" and still disagree about what your model is.** The format is a container, not a contract. Every DCC has its own internal mesh model, normal model, transform model, rig model, and material model, and export is a *translation* — with all the loss, ambiguity, and dialect that implies.

### 12.1 Why the same file behaves differently in two applications

**FBX is a container with many legal ways to say the same thing.**

- **Layer elements with mapping/reference modes.** Normals, UVs, vertex colours, tangents, and material IDs are each stored as a "layer element" with a *mapping mode* (`ByPolygonVertex`, `ByVertex`, `ByPolygon`, `ByEdge`, `AllSame`) and a *reference mode* (`Direct`, `IndexToDirect`). 3ds Max, Maya, Blender, and Houdini do not all pick the same combination, and importers do not all handle every combination. The usual symptom is **normals, vertex colours, or a secondary UV set silently arriving as defaults** — you get smooth-shaded-everything, white vertex colours, or a missing UV2.
- **Geometric transform** (`GeometricTranslation` / `GeometricRotation` / `GeometricScaling`). FBX has a node transform *and* a separate mesh-level offset that is not inherited by children. 3ds Max in particular uses it to represent an object whose pivot differs from its mesh. Blender applies it; Unity/Unreal bake or ignore it; some importers drop it entirely. Symptom: **the mesh appears offset from where it should be, or a child object is in the wrong place.**
- **Pre-rotation and post-rotation.** Maya's `jointOrient` becomes FBX `PreRotation`. Applications with no equivalent concept (Blender, which uses bone roll; most engines) must fold it into the rotation — which changes what the animation curves mean and makes a round trip non-identity.
- **Transform inheritance type** (`eInheritRrSs`, `eInheritRSrs`, `eInheritRrs`). This is how Maya's *segment scale compensate* survives. Almost nothing outside Maya respects it. Symptom: **a scaled joint scales its children when it shouldn't**, so limbs balloon or shrink during animation.
- **Unit and axis metadata.** `UnitScaleFactor`, `OriginalUnitScaleFactor`, `UpAxis`, `FrontAxis`, `CoordAxis`, and their signs. Exporters write these inconsistently and importers apply them inconsistently. If the exporter already converted the geometry *and* wrote the metadata, an importer that also converts gives you a **double conversion** — 100× scale errors and the notorious "rotated 90° on X, scaled by 100" root transform.
- **Version and encoding.** FBX 6100 ASCII, FBX 7.x binary (2011 / 2012 / 2013 / 2014-15 / 2016-17 / 2018 / 2019 / 2020...). Practical consequence: **Blender cannot import ASCII FBX at all** (it errors out), and cannot read pre-7.1 files. Old Max/Maya pipelines and many free asset libraries still ship ASCII FBX. Fix: re-export as binary 7.4+ from an Autodesk product.
- **Closed SDK vs reverse-engineered readers.** Autodesk products use the official FBX SDK; Blender, Godot, Assimp, three.js, and most open-source tools use reverse-engineered implementations. They are excellent but not identical, especially for edge-case layer elements, animation stacks, and constraints.
- **Animation stacks/takes, animation layers, and curve types.** Multiple takes are supported unevenly; animation layers usually must be flattened; curve tangent types (TCB, auto, custom) do not all survive.
- **Embedded media.** "Embed Media" packs textures into the FBX and unpacks them to a `.fbm` folder on import — or leaves the importer looking for absolute paths from another artist's machine.
- **Custom properties / user data** are storable but almost never read by the receiving application.

**Rule of thumb:** if a value is not part of the small common core (positions, indices, one UV set, one normal set, skin weights, joint transforms, baked keys), assume it will not survive.

### 12.2 Triangulation: the single most under-appreciated interchange problem

GPUs draw triangles. Everything else is triangulated *somewhere*. The question is **where, by whom, and with which diagonal** — and different applications answer differently.

**The common algorithms**

| Method | How it works | Where you see it | Failure modes |
|---|---|---|---|
| **Fan** | Pick vertex 0, connect it to every other vertex: (0,1,2), (0,2,3), (0,3,4)… | The naive/default in many exporters, converters, and older tools; historically the behaviour many 3ds Max users associate with FBX/OBJ export of polys | On **concave** n-gons it produces triangles *outside* the polygon and overlapping/inverted geometry; on non-planar quads it picks a diagonal arbitrarily; produces slivers on elongated n-gons |
| **Strip** | Zig-zag alternating vertices | Legacy hardware paths, some exporters | Same concavity problems; different diagonal than fan |
| **Shortest / fixed diagonal** | Always split the quad along the shorter (or always the 0-2) diagonal | Blender's "Fixed"/"Shortest Diagonal" quad methods, several engines | Deterministic but ignores shape quality; "Fixed" and "Fixed Alternate" give mirrored results |
| **Beauty / Delaunay-like** | Choose the split that maximises the minimum angle | Blender "Beauty", most modern remeshers | Best quality; **not** what most exporters default to, so it mismatches the engine |
| **Ear clipping** | Iteratively clip convex "ears" | Robust general-purpose n-gon handling | Handles concavity, but result depends on iteration order; fails on self-intersecting n-gons |
| **Monotone decomposition** | Split into monotone pieces, then triangulate | Robust library implementations | Different result again |
| **Hidden-edge / stored diagonal** | The application already stores a chosen diagonal per polygon and exports that | **3ds Max Editable Poly** keeps hidden edges / edge orientation; the FBX exporter has *Preserve edge orientation* and *Triangulate* options precisely for this | Only preserved if the receiving app honours the exported triangle list rather than re-quadding and re-splitting |

**Why the diagonal matters**

1. **Silhouette and shape change.** A non-planar quad is *ambiguous*. The two diagonals give two physically different surfaces. Hard-surface bevels, car panels, and any warped quad visibly change shape.
2. **Shading changes.** Normal interpolation runs per triangle. A different split means different interpolation, so highlights and gradients move.
3. **Normal-map bakes break.** This is the big one. If you bake in Substance 3D Painter/Marmoset against one triangulation and render in Unreal/Unity/Blender against another, the normal map is subtly wrong everywhere — you get faceting, dark corners, and "the bake looks fine in Painter but bad in engine". The industry-standard fix: **triangulate the low-poly once, upstream, and use that exact triangulated mesh for baking, texturing, and engine import.**
4. **Vertex/index order changes.** Which breaks blendshapes/morph targets, point caches, Alembic, per-vertex data, and any external tool that stores data by index.
5. **Vertex counts change**, so budgets and memory estimates change.
6. **Concave n-gons triangulated by fan produce inverted/overlapping triangles** — black shading, wrong normals, geometry sticking out through the surface.

**Practical policy**
- Model in quads, **export triangulated deliberately** (Triangulate in the exporter, or a Triangulate modifier/node applied), and never let two downstream tools triangulate independently.
- Never leave n-gons in a mesh that will be exported.
- Keep the quad version as the editable source; treat the triangulated version as a build artifact.
- If you must keep quads (e.g. for subdivision downstream), make sure every consumer subdivides rather than triangulates.

### 12.3 Application profiles

Each profile lists: coordinate/unit conventions → internal model quirks → what breaks when its files are opened elsewhere → what breaks when it opens other people's files.

---

#### 12.3.1 Autodesk 3ds Max

- **Conventions:** Z-up, right-handed. **System unit defaults to inches**, and display units are separate from system units — the two get confused constantly. "Generic units" means the file carries no real-world scale at all.
- **Mesh model:** Editable Poly stores polygons with *hidden edges* defining an internal triangulation; Editable Mesh is triangles only. Smoothing is expressed as **smoothing groups** (a 32-bit mask per face), not as hard/soft edges or split normals. Map channels are numbered (channel 1 = main UV, channel 2 commonly the lightmap channel, channel 0 and negative channels carry vertex colour/alpha/illumination).
- **Exporting FBX — the settings that bite:**
  - *Smoothing Groups* checkbox: **off by default in some presets.** With it off, receiving apps get no hard-edge information → everything imports fully smooth or fully faceted, and Unreal logs "no smoothing group information was found".
  - *Triangulate* / *Preserve edge orientation*: decides §12.2 for you.
  - *Tangents and Binormals*: requires UVs and triangulated geometry; if it fails, the receiver computes its own basis (usually fine if it uses MikkTSpace, wrong if not).
  - *Units*: "Automatic" combined with a non-standard system unit is the #1 cause of 100× / 2.54× scale errors.
  - *Up axis*: Max writes Y-up by default even though Max is Z-up, so the mesh is pre-rotated. Blender then adds its own conversion.
  - *Preserve instances*: off means every instance is exported as full duplicate geometry — file size explodes.
  - *Embed media*, *Bake animation*, *Resample*, *Deforming models*, *Skins*, *Morphs* all silently change what comes out.
- **Opening Max FBX elsewhere:**
  - **Smoothing groups become split normals / sharp edges.** Blender receives custom split normals; the *group identity* is gone, so a round trip back to Max loses the artist's smoothing-group organisation.
  - **Pivot offsets** ride in the FBX geometric transform → offset meshes in importers that ignore it.
  - **Multi/Sub-Object materials** become multiple material slots; material IDs that point past the end of the sub-material list produce missing/wrong assignments, and slot *order* may differ.
  - **V-Ray / Corona / Arnold / Physical materials** do not survive — you get a grey or diffuse-only approximation.
  - **Modifier stack** is baked; nothing parametric survives. Unapplied *Reset XForm* issues and mirrored objects with negative scale arrive with **inverted normals/winding**.
  - **Biped** rigs export as joints plus a lot of helper nodes; the Figure-mode pose is the bind pose, and biped's rotation conventions confuse retargeting. CAT rigs similarly.
  - **XRefs, particle systems, MassFX, procedural objects** need to be collapsed first or they arrive empty.
  - Object names with spaces and duplicate names (Max permits both) break engines and Maya.
- **Opening others' files in Max:** custom split normals arrive as explicit normals (Edit Normals modifier) rather than smoothing groups — usable but awkward, and any subsequent operation that resets normals loses them. Blender-authored FBX often arrives at 0.01 or 100 scale.
- **Native `.max`:** proprietary, **not backward compatible** (a 2025 file will not open in 2023); "save as previous version" only goes back a couple of releases; third-party plugin objects appear as missing/placeholder for anyone without the plugin.

---

#### 12.3.2 Autodesk Maya

- **Conventions:** Y-up (configurable to Z-up, which then mismatches everyone else), right-handed, **internal linear unit is centimetres** — the source of the classic ×100 (to metres) and ÷100 confusion with Unity.
- **Mesh model:** quads/n-gons, hard/soft **edges** (not smoothing groups), multiple UV sets with names (`map1` is the default and its name matters to some tools), per-vertex-face normals, colour sets.
- **Rig model:** joints with a separate `jointOrient` attribute, `segmentScaleCompensate`, namespaces, references, construction history, deformers (skinCluster, blendShape, lattice, wrap, deltaMush).
- **Opening Maya FBX elsewhere:**
  - **`jointOrient` → PreRotation.** Blender has no equivalent and converts to bone roll + connected/disconnected bones with invented lengths; the rig will *look* fine and *round-trip* badly.
  - **`segmentScaleCompensate` → inheritance type**, ignored almost everywhere → children of scaled joints misbehave.
  - **Namespaces leak into node names** (`char_rig:l_arm_jnt`) and break engine naming conventions and retargeting.
  - **Arnold/aiStandardSurface and any shading network** do not export; only Lambert/Blinn/Phong/StingrayPBS approximations do.
  - **Blendshape in-betweens, combination targets, and deformer stacks** do not survive; only flat morph targets do.
  - **Construction history and non-deformer deformers must be baked**, or the mesh exports in its pre-deformed state.
  - **Freeze transforms / delete history** not done → hidden offsets in the exported transforms.
  - **N-gons and non-planar faces** are permitted by Maya and get triangulated by whoever receives them (§12.2).
- **Opening others' files in Maya:** Blender armatures arrive with an extra `Armature` node and non-Maya joint orientation; Max smoothing groups arrive as hard edges (acceptable); FBX with geometric transforms is applied to the shape node in ways that surprise riggers.
- **Native `.ma` / `.mb`:** `.ma` is ASCII (diff-able, recoverable, larger); `.mb` is binary (smaller, opaque, riskier to corrupt). Both are version-locked forward and depend on plugins (Arnold, Yeti, nCloth caches, third-party rigs) — open without the plugin and nodes are lost on save.

---

#### 12.3.3 Blender

- **Conventions:** **Z-up**, right-handed, metres. Its FBX exporter converts to Y-up for the outside world by writing a compensating transform, which is where "why is my object rotated -90° on X with scale 100 in Unreal?" comes from.
- **Mesh model:** n-gon capable BMesh; smoothing is per-edge sharp flags plus **custom split normals** (Auto Smooth pre-4.1, the *Smooth by Angle* modifier in 4.1+); multiple UV maps and colour attributes; modifiers are non-destructive until applied.
- **Rig model:** armatures with bones that have **head, tail, length and roll** — a fundamentally different model from Maya/Max joints, which are points with an orientation. Blender must invent bone lengths when importing, and re-export produces different orientations.
- **Exporting FBX — settings that bite:**
  - *Path Mode* (Auto/Relative/Absolute/Copy/Embed) — the cause of missing textures.
  - *Apply Scalings* (All Local / FBX Units Scale / FBX Custom Scale) and *Apply Transform* (marked experimental) — these interact with the receiver's own unit handling and are the main levers for the ×100 problem.
  - *Smoothing: Normals Only / Face / Edge* — the default (**Normals Only**) writes no smoothing-group data, which is exactly what triggers Unreal's "no smoothing group information was found" warning. Use *Face* or *Edge* when the target expects groups.
  - *Apply Modifiers* — off means the Subdivision/Mirror/Solidify result never leaves Blender; on means shape keys and modifiers can conflict (shape keys are dropped by some modifier combinations).
  - *Add Leaf Bones* — adds an extra terminal bone to every chain, which pollutes skeletons in engines. Usually turn it off.
  - *Primary/Secondary Bone Axis* — must match what the target expects or every bone is rotated.
  - *Bake Animation / Simplify* — key reduction introduces animation error.
- **Opening Blender FBX elsewhere:**
  - **Extra `Armature` object node** in the hierarchy → engines see an additional root bone/node.
  - **Bone roll and invented bone lengths** differ from the original DCC.
  - **No smoothing groups** unless explicitly enabled.
  - **Only trivially-simple materials export**: base colour, roughness, metallic, normal — and only when an Image Texture node is connected directly to the Principled BSDF. **All procedural/node-based materials are lost.** Bake to textures first.
  - **Geometry Nodes, particle systems, physics, drivers, constraints, and modifiers must be applied/baked** or they do not exist outside Blender.
  - **Scale/rotation not applied** (Ctrl+A) → hidden transforms downstream.
- **Opening others' files in Blender:**
  - **Cannot read ASCII FBX or FBX < 7.1** — a hard failure, not a degradation.
  - Maya/Max rigs import with mangled bone orientation (functional, ugly, bad for round-tripping).
  - Very large scenes (Revit/CAD FBX) import slowly and with thousands of objects and materials.
- **Native `.blend`:** forward-compatible-ish for opening old files, but **new files do not open in older Blender**; textures may be packed or linked; linked libraries break on path changes; add-on-generated data is lost without the add-on. Version-to-version changes (e.g. the 4.1 Auto Smooth removal) can alter shading of old files on load.

---

#### 12.3.4 Maxon Cinema 4D

- **Conventions:** Y-up, **left-handed** coordinate system (the odd one out), centimetres by default.
- **Quirks:** the left-handedness means naive conversions produce **mirrored geometry or inverted Z** in other apps. Smoothing is controlled by **Phong tags** with an angle, which map imperfectly onto smoothing groups/split normals.
- **What does not survive:** MoGraph cloners, effectors, and generators must be made editable ("Current State to Object") or they export as empty nulls; deformers must be baked; Redshift/Standard/Physical materials export only as a crude approximation; Takes; XPresso.
- **Alembic is the safer path** for C4D → other-app geometry and simulation caches.
- **Native `.c4d`:** version-locked, plugin-dependent (Redshift, X-Particles), effectively unreadable outside C4D.

---

#### 12.3.5 SideFX Houdini

- **Conventions:** Y-up, right-handed, metres.
- **Quirks:** Houdini is attribute-centric — point/vertex/primitive/detail attributes with arbitrary names. **FBX carries almost none of this.** Normals live in the `N` attribute at point or vertex level (cusp angle determines hardness); UVs are usually a vertex attribute named `uv` — exporters that expect `map1`/`UVMap` may miss them.
- **Packed primitives and instances** must be unpacked or the export is empty/incorrect.
- **Hierarchy** is expressed via a `path`/`name` attribute rather than a scene graph; export tools reconstruct it and often get it wrong.
- **Preferred interchange:** **USD and Alembic**, not FBX. Houdini's FBX support is the weakest part of its interchange story.
- **Native `.hip`:** the file is a procedural graph, not geometry — useless to anyone without Houdini and the same HDAs/licences (Apprentice files are also non-commercial-locked and cannot be opened in commercial builds).

---

#### 12.3.6 ZBrush

- **Conventions:** arbitrary internal scale; the document's units mean nothing until you set export scale/offset.
- **Quirks:** meshes are tens of millions of polygons — most receivers cannot open a raw export. Subtools are separate objects; polygroups are a ZBrush-only concept; **polypaint is per-vertex colour** and only meaningful at very high density (must be converted to a texture). UVs are frequently absent.
- **Displacement / vector displacement maps** carry tangent-space and channel-orientation conventions (flip V, flip green, 32-bit vs 16-bit, scale/midpoint) that must be matched by the renderer or the detail comes out inverted or exploded.
- **GoZ** transfers work between supported apps but is a black box; misconfiguration silently overwrites meshes.
- **Best practice:** decimate or retopologise before export; bake detail to maps; export the *low-poly* to the rest of the pipeline.
- **Native `.ztl` / `.zpr`:** ZBrush-only.

---

#### 12.3.7 Adobe Substance 3D Painter / Designer

- **The triangulation contract:** Painter triangulates the mesh on import and bakes against that triangulation. If the engine triangulates differently, **every normal map is subtly wrong** (§12.2). Import the already-triangulated mesh.
- **Texture set = material/mesh name.** Renaming meshes or materials between the baking mesh and the final mesh orphans texture sets.
- **Normal-map format** (OpenGL vs DirectX) must be set to match the target engine, and the export preset must match the target's channel packing.
- **UDIM** projects do not export cleanly to engines that expect a single tile.
- **High-poly/low-poly name matching** (`_high`/`_low` suffixes) drives bake pairing; wrong names cause cross-baking artifacts.
- Bakes contain per-project settings (cage, max distance, anti-aliasing) that are not stored in the exported textures — reproducibility depends on keeping the project.

---

#### 12.3.8 Trimble SketchUp

- **Conventions:** inches internally; Z-up.
- **Fundamental mismatch:** SketchUp models **surfaces, not solids**. Faces have a **front and a back with separate materials**, reversed faces are extremely common and invisible in SketchUp's default display, and "solids" are only solid if the group happens to be watertight.
- **On export:** reversed faces become **inverted normals** (black or invisible surfaces elsewhere, and unprintable geometry); back-face materials are lost; textures applied with SketchUp's projective/distorted mapping produce strange UVs; everything triangulates; groups/components may flatten or explode into thousands of objects; no thickness means nothing can be printed or simulated without adding it.
- **Native `.skp`:** version-locked; readable by a limited set of third-party importers.

---

#### 12.3.9 Rhino / Grasshopper

- **Conventions:** user-selectable units; Z-up; **absolute tolerance** (default ~0.001) governs whether surfaces stitch into solids.
- **Quirks:** Rhino models NURBS. Exporting to mesh formats runs a **render-mesh tessellation** whose settings (density, max angle, max aspect ratio, min edge length) entirely determine the output quality — the default is often too coarse for close-ups and too fine for real-time.
- **Open polysurfaces / naked edges** are extremely common (a "solid-looking" object that is not closed) → fails printing, booleans, CFD.
- **Blocks (instances)** flatten on export; **layers double as material assignment**, so material data arrives as layer names.
- **Grasshopper definitions** are procedural and produce nothing until baked.

---

#### 12.3.10 Mechanical CAD (SolidWorks, Fusion 360, Inventor, CATIA, NX, Creo, Onshape)

- **Conventions:** millimetres or inches, Z-up, exact BREP.
- **Exporting to DCC/real-time:**
  - Everything is **tessellated** on the way out; the tessellation settings are the whole ballgame (§2.12). Defaults typically give either visibly faceted cylinders or absurd triangle counts.
  - **No UVs at all** — CAD has no texture parameterisation. Every imported CAD mesh needs unwrapping or triplanar shading.
  - **No smoothing information** beyond what the tessellator emits; you often need to recompute normals with an angle threshold.
  - **Per-face colours and appearances** survive inconsistently (STEP AP242 and 3D PDF handle them better than AP203).
  - **Assembly structure** may flatten into one mesh or explode into thousands of tiny parts, each a separate draw call.
  - **Part names** become long, duplicated, and full of illegal characters.
  - Internal/hidden geometry (screw threads, internal channels) is exported too, wasting enormous budget — **defeature first**.
- **STEP/IGES into CAD:** dumb solids with no feature tree; healing required; IGES commonly arrives as unstitched surfaces; tolerance mismatch between kernels produces gaps.
- **Native `.sldprt` / `.ipt` / `.f3d` / `.catpart`:** kernel- and version-locked; assemblies carry external references that break when files move.

---

#### 12.3.11 BIM (Revit, Archicad, Navisworks, IFC)

- **Conventions:** feet-and-inches or metric; real-world **survey coordinates**, which are often hundreds of thousands of units from the origin → immediate float-precision problems (§3.3) in any DCC or engine. Always relocate to a project base point before export.
- **Exports** (usually FBX to 3ds Max, or IFC) produce: tens of thousands of objects, thousands of near-duplicate materials with generated names, no useful UVs, inconsistent normals, and geometry at LOD far higher than needed for visualisation.
- **IFC** carries semantics (this is a wall, this is fire-rated) that every mesh-based tool discards; exporter quality varies enormously between authoring tools.
- **Round-tripping is effectively impossible** — BIM→DCC is a one-way trip.

---

#### 12.3.12 Marvelous Designer / CLO

- Outputs **dense, uniformly triangulated** garment meshes with pattern-derived UVs.
- Welded vs unwelded seams change topology and vertex count; the "thin"/"thick" export option changes whether you get a single surface or a shelled solid.
- No rig, no LODs; needs retopology and re-skinning for real-time.
- Simulation caches export as Alembic/point cache — huge, and tied to the exact mesh they were made for.

---

#### 12.3.13 Game engines as importers

**Unreal Engine** — Z-up, **left-handed**, 1 unit = 1 cm.
- Import options *Convert Scene*, *Force Front XAxis*, *Convert Scene Unit* change orientation and scale; wrong combination = model on its side or 100× off.
- *Import Normals* vs *Compute Normals* vs *Import Normals and Tangents* — computing normals discards the artist's custom normals; importing tangents without a matching basis reintroduces §4.2 problems (Unreal uses MikkTSpace when computing).
- Warns on missing smoothing groups (see Blender, §12.3.3).
- Naming conventions matter: `UCX_`/`UBX_`/`USP_` for collision, `_LOD0..n` for LODs, sockets, `SM_`/`SK_` prefixes.
- Skeletal meshes must match an existing skeleton asset or a new one is created — the classic cause of "my animations don't work on my character".
- Morph targets require *Import Morph Targets* and a stable vertex order.
- Material slots map by name/order; changing them re-links or unlinks materials on reimport.

**Unity** — Y-up, **left-handed**, 1 unit = 1 m.
- *Scale Factor* / *Convert Units* handle Maya/Max centimetres; `.blend` files are imported by invoking Blender's FBX exporter in the background, so all Blender export caveats apply invisibly.
- *Normals: Import/Calculate/None*, *Smoothing Angle*, *Tangents: Import/Calculate (MikkTSpace)/None*.
- Humanoid avatar mapping requires a recognisable bone structure and T-pose; failures are opaque.
- *Read/Write Enabled*, mesh compression, and optimisation settings change vertex order and can break external references.
- Legacy vs modern blendshape normal handling causes lighting differences on morphs.

**Godot / three.js / Babylon.js / Sketchfab / web viewers** — **prefer glTF**. FBX support is partial or via conversion. glTF is triangles-only, Y-up, right-handed, metres, PBR metal-rough, single-index-buffer — so it forces resolution of every ambiguity in §1.2, §12.1 and §12.2 at export time, which is exactly why it is the most reliable interchange target for real-time.

---

### 12.4 Common migration paths and what specifically breaks

| Path | Watch for |
|---|---|
| **3ds Max → Blender (FBX)** | Smoothing groups → custom split normals (group identity lost); Z-up double conversion; system-unit-driven scale error; pivot via geometric transform; V-Ray/Corona materials gone; instances duplicated; negative-scale mirrors arrive inside-out; ASCII FBX from old pipelines simply refuses to import |
| **Blender → 3ds Max (FBX)** | Extra Armature node; no smoothing groups unless enabled; procedural materials gone; unapplied modifiers/transforms; leaf bones; 0.01/100 scale |
| **Maya → Blender (FBX)** | jointOrient → bone roll (rig round-trips badly); namespaces in names; cm→m scale; Arnold materials gone; blendshape in-betweens gone; segment scale compensate ignored |
| **Blender → Maya (FBX)** | Bone lengths/rolls foreign to Maya; extra armature node; Z-up conversion; material approximation only |
| **Blender → Unreal** | ×100 scale + −90° X rotation; "no smoothing group information" warning; leaf bones; extra root; use *Apply Transform* or set units to cm and export with the right axis settings |
| **Maya/Max → Unity** | cm vs m (Scale Factor 0.01); left-handed conversion flipping mirrored parts; humanoid rig mapping failures |
| **Cinema 4D → anything** | Left-handed → mirrored/flipped Z; MoGraph and deformers must be made editable; prefer Alembic |
| **Houdini → anything (FBX)** | Attribute loss; packed prims; hierarchy reconstruction; prefer USD/Alembic |
| **ZBrush → Substance → Engine** | Poly count; missing UVs; polypaint needs baking; triangulation must be locked before baking; displacement channel/flip conventions |
| **Substance Painter → Engine** | Normal-map green channel (OpenGL vs DirectX); channel packing preset; texture-set naming; triangulation match |
| **CAD (STEP/SolidWorks) → DCC** | Tessellation density; no UVs; no smoothing; internal geometry; thousands of parts; mm vs m; part naming |
| **SketchUp → anything** | Reversed faces / inverted normals; no thickness; front/back materials; inches; exploded groups |
| **Revit/BIM → DCC/Engine** | Survey-coordinate offset destroying precision; object and material explosion; no UVs; one-way trip |
| **Rhino → mesh pipeline** | Render-mesh tessellation settings; open polysurfaces; blocks flattened; layer-as-material |
| **Anything → 3D printing** | Units (STL is unitless, slicers assume mm); normals; watertightness; tessellation coarseness (§13.3) |
| **Anything → Web/glTF** | Triangles only; single UV set commonly assumed; metal-rough only; no smoothing groups concept; Draco/meshopt compression changes vertex data |

### 12.5 Native project formats: what "opening it in other software" really means

| Format | Openable elsewhere? | Main risks |
|---|---|---|
| `.max` | No (some viewers/converters only) | Not backward compatible; plugin-object dependency; save-as-previous only spans a few versions |
| `.ma` / `.mb` | No | Version-locked; plugin nodes lost on save-without-plugin; `.mb` corruption is unrecoverable |
| `.blend` | No | New files won't open in older Blender; add-on data lost; linked libraries break on move; version behaviour changes (e.g. Auto Smooth) can alter shading |
| `.c4d` | No | Version- and plugin-locked |
| `.hip` | No | It's a graph, not geometry; HDA and licence dependencies |
| `.ztl` / `.zpr` | No | ZBrush-only |
| `.3dm` (Rhino) | Partially (openNURBS is open, so several apps read it) | Tessellation still required; version differences |
| `.skp` | Partially | Version-locked; surface-not-solid model |
| `.sldprt` / `.ipt` / `.catpart` | No (viewers/converters only) | Kernel/version lock; external references |
| `.rvt` | No | Enormous; one-way to everything else |

**The universal rule:** a native project file is a *working file*, not a *deliverable*. Anything that must cross an application boundary needs an explicit interchange export, with the conventions agreed in advance.

### 12.6 Choosing the interchange format

| Need | Best choice | Why |
|---|---|---|
| Static mesh to a real-time engine or the web | **glTF/GLB** | Well-specified, PBR-native, no ambiguity, compressible |
| Rigged character + animation between DCCs and engines | **FBX** (with agreed settings) | Still the only broadly-supported option; USD is catching up |
| Baked deforming geometry / simulation caches | **Alembic** | Robust, topology-stable, no rig baggage |
| Large scenes, layered collaboration, film pipelines | **USD** | Composition, referencing, non-destructive layering |
| Volumes | **OpenVDB** | Purpose-built |
| Exact engineering geometry | **STEP (AP242)** | BREP + colours + PMI |
| 3D printing | **3MF** (fall back to STL) | Units, colour, materials, manifold-aware |
| Quick static geometry | **OBJ** | Universal, but no rig/anim/units |
| Materials across renderers | **MaterialX** (+ USD) | The only real attempt at portable shading |

### 12.7 Pre-export checklist (application-agnostic)

- [ ] Units and system unit confirmed **on both ends**, and exporter unit option chosen deliberately (not "Automatic" on faith)
- [ ] Up axis and forward axis agreed; verified by importing a known-asymmetric test object
- [ ] Transforms frozen/applied; no negative or non-uniform scale; pivots correct
- [ ] Mesh **explicitly triangulated** (or explicitly kept as quads with all consumers subdividing)
- [ ] No n-gons; no non-planar quads where shape matters
- [ ] Smoothing exported in the form the target expects (smoothing groups vs split normals vs sharp edges)
- [ ] Modifiers/deformers/generators applied or baked; instances handled deliberately
- [ ] Materials baked to textures if the target cannot read the shading graph
- [ ] Texture paths relative or embedded; no absolute paths, no spaces, correct case
- [ ] Names sanitised: no duplicates, no spaces, no non-ASCII, namespaces stripped, target naming conventions applied
- [ ] Rig: bind pose verified, influence count within target limit, weights normalised, leaf/extra bones handled, joint orientation convention matched
- [ ] Animation: frame rate, time range, baked keys, resample setting, root motion convention
- [ ] FBX written as **binary 7.x**, at a version the receiver supports
- [ ] Export re-imported into a *third* application as a sanity check before hand-off
- [ ] Export settings saved as a shared preset and committed to the project, not left to each artist

### 12.8 Interchange symptom → cause

| Symptom after opening the file elsewhere | Likely cause |
|---|---|
| Model 100× / 0.01× / 2.54× wrong size | Unit metadata applied twice, or system unit ≠ display unit (Max), cm vs m (Maya/Unity) |
| Model lying on its side or rotated −90° X | Up-axis conversion applied by both exporter and importer |
| Object offset from its pivot / children misplaced | FBX geometric transform ignored |
| Whole model fully smooth or fully faceted | Smoothing groups not exported (Max/Blender setting) or layer element mapping unsupported |
| Shading looks right in the source app, subtly wrong after import | Triangulation mismatch, or tangent basis recomputed differently |
| Normal map looks inverted | Green-channel convention (OpenGL vs DirectX) |
| Some faces invisible or black | Negative scale/mirroring flipping winding; SketchUp reversed faces |
| Materials all grey/default | Renderer-specific shading network (V-Ray, Arnold, Redshift, procedural nodes) not exportable |
| Missing textures | Absolute paths, no embed, case sensitivity, `.fbm` folder not shipped |
| Character explodes on import | Bind pose / inverse-bind mismatch, axis conversion, influence overflow |
| Limbs balloon during animation | Segment scale compensate / inheritance type ignored |
| Extra root bone or extra node in hierarchy | Blender's Armature object, or leaf bones |
| Blendshapes/morphs broken or missing | Vertex order changed by triangulation or optimisation; in-betweens unsupported |
| File won't open at all | ASCII FBX into Blender; FBX version newer than the importer; native format from a newer app version |
| File is enormous / scene has 40,000 objects | Instances exported as duplicates; CAD/BIM export without defeaturing or grouping |
| Object hundreds of kilometres from the origin, jittering | BIM/GIS survey coordinates not rebased |

---

## 13. Domain-Specific Problems

### 13.1 Real-time games

- Triangle, draw call, texture memory, and bone budgets per platform.
- Collision authoring: convex decomposition quality, simple primitives, complex-vs-simple collision, collision not matching visuals.
- Navigation mesh generation failures from non-manifold or overly detailed geometry.
- Modular kit alignment: grid snapping, seams between modules, lighting discontinuities, tiling texture alignment.
- Foliage: alpha-test overdraw, TAA behavior, wind vertex animation breaking motion vectors and bounds, two-sided lighting.
- Hair cards: sorting, depth, alpha, aliasing.
- Skinned mesh bounds and shadow-only proxies.
- Platform variance: mobile precision, console memory, PC driver differences, WebGL/WebGPU limits.
- Determinism for replays and lockstep networking.
- Asset streaming, hitching, and shader PSO precompilation.

### 13.2 Film and VFX

- Scene scale consistency across departments (a single wrong scale invalidates lighting, sim, and DOF).
- Subdivision at render time — the render mesh is not the file mesh; displacement bounds and dicing rates control memory and can blow up a farm node.
- Displacement bound errors → chopped-off geometry.
- Render times and farm cost; per-frame variance; out-of-core geometry.
- Deep compositing and holdout/matte correctness.
- Motion vectors for comp-side blur; correct for deformation, incorrect for topology changes.
- Camera solve/matchmove: lens distortion, sensor size, focal metadata, survey scale.
- Color pipeline (ACES/OCIO) consistency from plate to delivery.
- Asset versioning and shot publishing; broken references at scale.
- Crowd systems: instancing, variation, memory.
- USD composition conflicts across departments.

### 13.3 3D printing / additive manufacturing

- **Watertight, manifold, correctly-oriented normals** — the hard requirement.
- **Multiple disconnected shells** — intended (multi-part) or accidental (stray geometry).
- **Wall thickness below the process minimum** → parts fail to print or crumble. Minimums differ per process (FDM ~0.8 mm, SLA ~0.5 mm, SLS ~0.7 mm — process and material dependent).
- **Minimum feature size and detail resolution** — text and fine embossing below nozzle/laser resolution disappear.
- **Overhangs and support** — angles beyond ~45° need supports; supports leave surface scarring and are hard to remove from internal cavities.
- **Bridging limits.**
- **Trapped volumes** in resin printing — hollow parts need drain holes or they cup and fail.
- **Orientation and anisotropy** — FDM parts are far weaker along the layer axis; orientation determines strength, surface finish, support need, and print time.
- **Warping, shrinkage, and thermal distortion** — dimensional accuracy differs from the model; may need scale compensation.
- **Tolerance and clearance for assemblies** — parts modeled at exact nominal dimensions will not fit; typical clearances 0.2–0.5 mm depending on process.
- **Units** — STL is unitless; the slicer assumes mm. A model authored in inches prints 25.4× too small.
- **Faceting from coarse STL export** — visible polygonal facets on curved surfaces; export with a fine chord tolerance.
- **File size explosion** from over-tessellation.
- **Self-intersections and inverted normals** → slicer produces inside-out or hollow-in-the-wrong-place results.
- **Zero-thickness surfaces** — a "solid-looking" surface model has no volume to print.
- **Non-planar bottom / no flat base** — adhesion failure.
- **Slicer-specific issues** — infill, seam placement, retraction/stringing, first-layer calibration, elephant's foot.
- **Multi-material/color** — STL cannot carry it; use 3MF/AMF/OBJ+MTL.

### 13.4 CAD, engineering and manufacturing

- BREP vs mesh mismatch; mesh imports arrive as un-editable "dumb solids".
- Import healing: gaps, slivers, short edges, tangent-face merging, tolerance normalization.
- **Topological naming problem** — features referencing edges/faces by index; upstream edits break downstream features.
- Over/under-defined sketches; rebuild errors; circular references.
- Assembly mate/constraint failures, interference and clearance checking, degrees-of-freedom errors.
- Kernel differences (Parasolid, ACIS, CGM, OpenCascade) causing translation loss.
- **Class-A surfacing** — curvature continuity, highlight/zebra-stripe analysis, draft-angle validation.
- Manufacturing checks: draft angles for molding, undercuts, minimum radii for machining, tool access, sheet metal unfold failures.
- **GD&T and tolerance stack-up** — models are nominal; drawings and PMI carry tolerances; MBD requires the PMI to survive translation.
- Large assembly performance; simplified representations and defeaturing.

### 13.5 Simulation: FEA, CFD, and analysis

- **Mesh quality metrics**: aspect ratio, skewness, orthogonal quality, Jacobian ratio, warpage, taper, minimum/maximum angle. Poor values → ill-conditioned systems, non-convergence, wrong results *that still look plausible*.
- **Element type and order** — tet vs hex vs prism, linear vs quadratic; linear tets are notoriously stiff.
- **Dirty geometry and defeaturing** — small fillets, holes, and logos generate millions of elements for no analytical value; but over-defeaturing removes real stress concentrations.
- **Watertight and non-self-intersecting** input required for volume meshing and CFD.
- **Boundary layer / inflation layers** in CFD — collapse in concave corners.
- **Mesh independence study** — results must converge as mesh refines; skipping this is the most common analysis error.
- **Contact and interface meshing** mismatch between parts.
- **Units and material property consistency** (the single most common source of wrong-by-1000× results).
- **Singularities** — sharp reentrant corners give stresses that diverge with refinement and never converge.

### 13.6 AR / VR / XR

- Strict frame-time budgets (72–120 Hz, doubled for stereo); dropped frames cause discomfort.
- **Scale accuracy** — VR is 1:1; scale errors are immediately, viscerally wrong.
- Comfort: motion sickness from latency, wrong IPD, incorrect world scale, vection, and camera motion the user didn't initiate.
- Stereo-specific artifacts: screen-space effects that don't match between eyes, billboards and impostors that break stereo, incorrect depth for UI.
- Foveated rendering artifacts.
- Mobile/standalone headset thermal throttling and memory limits.
- AR: real-world occlusion, lighting estimation mismatch, plane detection drift, anchor drift, scale/geo-alignment.
- WebXR asset size and load-time constraints.

### 13.7 Web and mobile 3D

- Download size — every megabyte costs conversion; use Draco/meshopt compression, KTX2/Basis textures.
- Decompression CPU cost and main-thread blocking.
- WebGL/WebGPU capability variance, mediump precision, texture format support.
- Memory ceilings on mobile browsers; context loss.
- Progressive loading and LOD streaming.
- Correct color management in browsers (often overlooked, producing washed-out PBR).

### 13.8 Photogrammetry, scanning and reality capture

- **Reflective, transparent, and featureless surfaces** fail to reconstruct (glass, chrome, white walls, water).
- **Noise and holes** in point clouds; occluded regions never captured.
- **Registration/alignment drift** across scans; loop closure errors; doming in aerial photogrammetry without control points.
- **Scale ambiguity** — photogrammetry has no inherent scale; needs a reference object, GCPs, or known baseline.
- **Baked lighting and shadows in textures** — the capture records the lighting of the day; delighting is imperfect and hard.
- **Moving objects** produce ghosting.
- **Enormous poly counts** (tens of millions) requiring decimation and retopology.
- **Bad topology** — the output is unusable for animation without retopology.
- **Georeferencing and coordinate systems** — CRS/EPSG mismatch, geoid vs ellipsoid height, projection distortion, huge coordinate values causing float precision failure.
- **LiDAR-specific**: intensity vs color, classification errors, multi-return handling, registration to imagery.
- **Privacy and legal** — faces, license plates, private property, protected sites.

### 13.9 Medical, scientific and volumetric

- Segmentation errors (over/under-segmentation) propagate directly into geometry.
- Anisotropic voxels (thin in-plane, thick slice spacing) produce stretched, stair-stepped surfaces.
- **Marching cubes artifacts** — staircase terracing, ambiguous cases producing holes, non-manifold output from some variants.
- Smoothing removes clinically relevant detail.
- Units and DICOM metadata (spacing, orientation, patient coordinate system).
- Regulatory validation for surgical planning and printed guides.
- Privacy/PHI in scan data and in reconstructed faces.

### 13.10 GIS, BIM and architecture

- Coordinate reference systems, projections, datums; large-coordinate float precision.
- Vertical datum mismatch (ellipsoid vs geoid vs local).
- LOD standards (CityGML LOD0–4) and semantic vs geometric detail.
- IFC semantic loss on export; property sets dropped.
- Model federation conflicts and clash detection producing thousands of false positives from tolerance settings.
- Enormous scene sizes; needs tiling (3D Tiles), streaming, and instancing.
- Units in architectural CAD (feet-and-inches vs metric) and drawing-scale assumptions.

### 13.11 Robotics, digital twins and simulation-to-real

- URDF/SDF/USD-Physics: visual mesh vs collision mesh separation; using visual meshes for collision destroys performance.
- Convex decomposition quality (V-HACD) for concave parts.
- Inertia tensors, mass, and center of mass — often wrong, causing implausible dynamics.
- Joint limits, axes, and frame conventions (ROS uses Z-up, X-forward; many assets don't).
- Mesh origin vs link frame mismatch.
- Sim-to-real gap: friction, restitution, sensor noise, and rendering domain gap for vision models.

### 13.12 AI-generated and neural 3D

- Text/image-to-3D output is often non-manifold, self-intersecting, poorly UV'd, and with baked lighting in textures.
- Janus problem (multi-faced artifacts), floaters, and low geometric fidelity.
- NeRF/Gaussian splats: no usable mesh, no collision, no relighting, view-dependent artifacts, floaters, enormous memory, no editability, poor topology after extraction.
- Training-data provenance and licensing uncertainty.
- Non-deterministic outputs complicating pipelines and review.

---

## 14. Cross-Cutting Concerns

- **Reproducibility** — the same input should produce the same output. Threading, GPU drivers, and random seeds break this.
- **Automation and validation** — manual QA does not scale; build automated validators into import/export (see §16).
- **Tolerance culture** — every tool has an epsilon; when two tools disagree about it, geometry "fails to stitch", "isn't watertight", or "has duplicate vertices" for no apparent reason. Standardize it.
- **Source of truth** — one canonical representation; derived assets are build outputs, never hand-edited.
- **Scale discipline** — define the project unit in writing, on day one.
- **Naming discipline** — enforced by tooling, not by hope.
- **Documentation of conventions** — up axis, unit, texel density, tangent basis, normal-map green channel, color space, LOD naming, pivot conventions. Most cross-team 3D bugs are convention bugs.
- **Accessibility and perception** — photosensitivity from flicker/strobing, color-only encoding of information, VR comfort options.
- **Sustainability/cost** — render farm and GPU cost of brute-force approaches.

---

## 15. Diagnostic Triage Table

| Symptom | Most likely causes |
|---|---|
| Model invisible | Wrong scale (too big/small), behind camera, flipped normals + backface culling, zero-scale transform, NaN bounds, wrong layer/visibility |
| Model inside-out | Flipped normals / inconsistent winding / negative scale |
| Flickering surfaces | Z-fighting from coplanar or duplicate geometry, depth precision, far-from-origin |
| Black patches on the mesh | Degenerate faces → NaN normals/tangents, missing UVs, failed bake, zero-area UV triangles |
| Faceted look | Missing/incorrect smoothing, lost custom normals, coarse tessellation, low-bit normal map |
| Dents instead of bumps | Normal map green-channel convention (OpenGL vs DirectX) |
| Shading seams along UV borders | Tangent basis mismatch, split normals at seam, insufficient padding |
| Washed-out or dark textures | sRGB/linear color-space tagging error, double gamma |
| Blurry in some places, sharp in others | Non-uniform texel density / UV stretching |
| Texture bleeding between islands | Insufficient UV padding, mip bleed |
| Character explodes into spikes on import | Bind pose / inverse-bind-matrix mismatch, wrong axis conversion, bone influence overflow |
| Limbs collapse when twisting | Candy-wrapper LBS artifact — needs twist joints or DQS |
| Animation too fast/slow | Frame-rate or time-unit mismatch |
| Objects jitter far from origin | Float32 precision — needs origin rebasing |
| Boolean fails or produces garbage | Non-manifold, self-intersecting, open, or coplanar input |
| Slicer says "not watertight" | Boundary edges, non-manifold edges, flipped normals, zero-thickness surfaces |
| Cracks between adjacent meshes | T-junctions, quantized vertex positions, LOD mismatch, floating point |
| FEA won't converge | Poor element quality (skewness/Jacobian), singularities at sharp corners, unit inconsistency |
| Shadows detached from objects | Excessive shadow bias (peter-panning) |
| Stripey self-shadowing | Shadow acne — insufficient bias / resolution |
| Light coming through walls | Zero-thickness geometry, lightmap leak, probe interpolation across walls |
| Ghosting/smearing in motion | TAA/upscaler with missing or wrong motion vectors |
| Terrible framerate with modest triangle count | Overdraw, draw calls, translucency, tiny triangles, texture memory thrash |
| Materials all grey/pink on import | Broken texture paths, unsupported material graph, missing shader |
| Different look in engine vs DCC | Color management, tone mapping, tangent basis, triangulation, unit scale |

---

## 16. Validation Checklists

### 16.1 Universal mesh sanity

- [ ] No NaN/Inf vertex positions
- [ ] No zero-area faces, zero-length edges, duplicate faces
- [ ] No unreferenced/loose vertices or edges
- [ ] Consistent winding, outward-facing normals
- [ ] No non-manifold edges or bowtie vertices (if solidity is required)
- [ ] No unintended boundary edges (holes)
- [ ] No self-intersections (if solidity is required)
- [ ] Acceptable triangle aspect ratio / minimum angle
- [ ] No unintended coincident/duplicate shells or interior faces
- [ ] Transforms applied/frozen; no unintended negative or non-uniform scale
- [ ] Correct pivot/origin, object centered as intended
- [ ] Correct unit scale, up axis, and forward axis
- [ ] Reasonable coordinate magnitudes (near origin)
- [ ] Sensible names; no duplicates or illegal characters

### 16.2 Texturing and materials

- [ ] All required UV sets present and in the right channel order
- [ ] No degenerate UV faces; no unintended overlaps
- [ ] Consistent texel density; UVs within the expected range
- [ ] Sufficient padding for the target resolution and mip count
- [ ] Color maps sRGB; data maps linear
- [ ] Correct channel packing convention
- [ ] Correct normal-map green channel and tangent basis (MikkTSpace)
- [ ] Correct compression format per map type
- [ ] Textures power-of-two (where required), mips generated
- [ ] Relative texture paths, correct case, no spaces
- [ ] Albedo within plausible range; no baked lighting in albedo
- [ ] Material count within budget

### 16.3 Rigged/animated assets

- [ ] Skeleton naming matches the target convention
- [ ] Bind pose matches inverse bind matrices
- [ ] Weights normalized; influences within the engine limit
- [ ] No stray weights; symmetric where expected
- [ ] Joint count within limit; no unsupported scale usage
- [ ] Deformation tested through full range of motion
- [ ] Blendshape vertex order preserved; names and order intact
- [ ] Animation frame rate and time range correct; loops seamless
- [ ] Root motion authored as the engine expects
- [ ] Constraints/IK baked as required

### 16.4 Real-time delivery

- [ ] Triangle, vertex, draw call, material, and bone counts within budget
- [ ] LODs present, with acceptable transitions
- [ ] Collision meshes present, simplified, and correctly named
- [ ] Lightmap UVs valid (non-overlapping, padded, in 0–1)
- [ ] Explicitly triangulated
- [ ] Bounding volumes correct (including for vertex-animated meshes)
- [ ] Texture memory within budget; streaming behavior verified

### 16.5 3D printing

- [ ] Watertight, manifold, outward normals, single (or intentional) shell
- [ ] No self-intersections
- [ ] Wall thickness above process minimum everywhere
- [ ] Feature sizes above process resolution
- [ ] Overhangs assessed; supports planned; drain holes for hollow resin parts
- [ ] Correct units (mm) and real-world scale verified
- [ ] Clearances added for mating parts
- [ ] Tessellation fine enough to hide facets, coarse enough for a sane file size
- [ ] Orientation chosen for strength and finish

### 16.6 CAD/simulation

- [ ] Solid (not surface) bodies where required; no gaps above tolerance
- [ ] No sliver faces or micro-edges
- [ ] Model tolerance consistent with the downstream tool
- [ ] Defeatured appropriately; stress-relevant features retained
- [ ] Mesh quality metrics within solver thresholds
- [ ] Mesh independence verified
- [ ] Units and material properties consistent throughout

---

## 17. Glossary

**Aspect ratio** — ratio of longest to shortest element dimension; a mesh-quality metric.
**BREP** — boundary representation; exact CAD surfaces plus topology.
**Bowtie vertex** — non-manifold vertex where two surface fans meet at one point.
**Cage** — offset surface controlling ray distance during normal-map baking.
**Cap triangle** — a triangle with one angle approaching 180°.
**Chord tolerance** — max deviation between a curved surface and its tessellated approximation.
**Coplanar** — lying in the same plane; the classic z-fighting setup.
**Degenerate geometry** — primitives with zero area/length/volume.
**Draw call** — one command to the GPU to render a batch; a real-time cost driver.
**Fan triangulation** — splitting a polygon by connecting one vertex to all others; fast, but invalid on concave n-gons.
**Geometric transform** — FBX's mesh-level offset separate from the node transform; commonly ignored by importers, causing offset meshes.
**Extraordinary vertex / pole** — a vertex whose valence differs from the regular case (4 for quads); source of curvature discontinuity.
**Firefly** — an isolated over-bright pixel from a rare high-energy light path.
**Gimbal lock** — loss of a rotational DOF when Euler axes align.
**Hidden edge** — 3ds Max's stored internal diagonal of a polygon; determines its triangulation.
**Inverse bind matrix** — transform from mesh space to bone space at bind time.
**Jacobian ratio** — element distortion metric in FEA.
**Layer element (FBX)** — how FBX stores normals/UVs/colours, with a mapping mode and reference mode; mismatched support causes silent attribute loss.
**Manifold** — every point has a disk-like neighborhood; the precondition for "solid".
**MikkTSpace** — the standard tangent-space generation algorithm.
**N-gon** — a polygon with five or more sides.
**Non-uniform scale** — different scale factors per axis; breaks normals, physics, and skinning.
**Namespace (Maya)** — a name prefix that leaks into exported node names.
**Overdraw** — shading the same pixel multiple times; the main cost of transparency and foliage.
**Peter-panning** — a shadow detached from its caster due to excessive bias.
**Segment scale compensate** — Maya's option stopping a joint's scale propagating to children; carried by FBX inheritance type and ignored almost everywhere else.
**Smoothing group** — 3ds Max's per-face bitmask defining which faces shade smoothly together; converted to split normals/sharp edges elsewhere.
**Sliver / needle triangle** — a long, thin triangle with poor aspect ratio.
**T-junction** — a vertex touching an edge it isn't topologically connected to; causes cracks.
**Tangent space** — the per-vertex TBN frame that makes normal maps work.
**Texel density** — texture pixels per unit of world-space surface area.
**Topological naming problem** — CAD features breaking when the topology they reference is renumbered.
**UDIM** — a tiled UV convention using multiple 0–1 tiles.
**Watertight** — closed, manifold, consistently oriented; encloses a volume.
**Winding order** — vertex ordering that defines a face's front side.
**Z-fighting** — depth-buffer precision conflict between near-coincident surfaces.

---

### How to use this document

1. **Before authoring**: agree conventions (unit, axis, texel density, color space, tangent basis, naming) and write them down.
2. **Before crossing an application boundary**: read §12 for the specific pair of tools involved, and agree conventions in writing.
3. **While authoring**: check §2 and §5 regularly, not at the end.
4. **Before export**: run the relevant checklist in §16, plus the pre-export checklist in §12.7.
5. **When something breaks**: start at §15 (general) or §12.8 (interchange-specific), then read the referenced section.
6. **When adopting a new domain**: read §13's subsection for that domain first — most domain-specific pain is domain-specific *requirements* you didn't know existed.
