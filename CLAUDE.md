# ArchVisu — Repository Guide

Monorepo root for **ArchVisu**: an automated pipeline that turns an arch-viz agency's
3D source scene (3ds Max + V-Ray/Corona, or SketchUp) into a mobile-optimized,
browser-based 3D walkthrough reachable from a URL + QR code — no app install.

Target market: Indian real estate / arch-viz agencies. Phase 1 = single building,
single flat. See `docs/` for the underlying research.

---

## 0. Current focus — read first

**Only the core transformation pipeline is being built right now:**

1. `exporters/maxscript/` — the 3ds Max extractor (Track 1 intake). Not started.
2. `pipeline/` — the headless Blender worker. Not started.
3. `contracts/` — the schemas those two agree on. Not started.

Everything else (`api/`, `dashboard/`, `viewer/`, `infra/`) is **deferred and
unscoped**. Do not scaffold them, and do not make design choices that presuppose them.

**Already built — two standalone instruments, neither of them the pipeline:**

`blender_core_processing_p0.1/` decimates a model by a flat ratio and serves a
browser HUD reporting load time, transfer size, and frame rate, so later
optimisation work has a measured reference rather than an estimate to beat. It
deliberately skips per-type ratios, UV preservation, and texture work.

`blender_core_processing_p0.2/` detects a building's storeys and rooms from
geometry alone, renders a cubemap at eye height in each, applies per-type
decimation with UV preservation, exports GLB, and serves a three.js viewer where
each capture point is a clickable hotspot. It implements §4's classification and
UV tiers for real, and adds the plan-dedup idea (identical storeys render once).
It does **not** do texture work — no ORM packing, no resize, no KTX2.

Both READMEs list every deviation. Do not grow either into `pipeline/` — start
that fresh. What they are for is measurements to replace the estimates in `docs/`.

Decisions deliberately left open — do not assume an answer:

- **Backend / job-queue language is undecided.** The pipeline must be runnable
  standalone from a CLI, with no queue, web framework, or service assumed. Wrapping
  it as a service comes later; keep that seam clean.
- **Storage provider is undecided** (AWS S3 vs Cloudflare R2). Put reads/writes
  behind a narrow storage interface. No provider SDK in pipeline logic.
- **SketchUp (Track 2) is Phase 2.** Its design is documented below so the contracts
  don't have to change when it arrives, but write no SketchUp code now. When
  designing `contracts/`, keep the GLB intake path representable — don't hard-code
  FBX-only assumptions into the schema.

---

## 1. Read this before trusting the spec documents

The five research documents in this repo were written at different times and
**they contradict each other**. `docs/ArchVisu-Requirements_v1.2.md` is the oldest
of the technical set; the three technical `.docx` reports are later and supersede
it on every point below. Do not implement from the requirements doc alone.

| Topic | Requirements v1.2 says | Later research says | **Authoritative** |
|---|---|---|---|
| SketchUp intake | FBX or GLB (FR-01) | FBX from SketchUp is broken — hierarchy and material assignments are lost | **GLB direct.** Never FBX from SketchUp. |
| Material conversion | Use V-RayMax Converter Pro ($45/license) + SketchUp converter (FR-02) | Agencies use drag-and-drop library materials = bitmaps in slots. A custom MaxScript that copies bitmap file paths handles 85–90%. | **Custom MaxScript extractor. Zero third-party plugin dependency.** |
| Texture baking | Bake all geometry after decimation (FR-04, FR-06) | 85–90% of materials are already flat PBR bitmaps. Baking them re-samples good data into worse data and costs 10–30x processing time. Cycles GPU baking is unreliable (Blender bug #59286). | **Conditional baking only** — procedural maps and flagged edge cases. Default path skips baking entirely. |
| UV handling | Auto-unwrap under a polycount threshold; above it → manual review (FR-06, FR-25) | UV quality tracks geometric complexity, not polycount. Most arch-viz meshes already have artist-authored UVs. | **Three-tier automated strategy** (§4). No polycount threshold. |
| Manual review queue | Blocking human step for Tier 3 materials + complex objects (FR-25) | Full automation is achievable; blocking humans doesn't scale | **No blocking human step.** Flag + preview state instead (§5). |
| Material tier mix | 60% / 25% / 15% (simple/moderate/complex) | 85–90% / 8–12% / 2–5% for library materials | **The revised numbers.** The pipeline is a file-management problem, not a shader-math problem. |
| Textures per material | Always emit all 5 maps, generating flat defaults (FR-19) | Emit only what exists; record `null` + status in metadata | **Emit what exists, mark nulls.** Blender applies defaults. Don't ship flat grey PNGs over Indian upload bandwidth. Delivery side packs to 3 maps via ORM (§4). |

When a task touches any row above, follow the "Authoritative" column and say so.
If you find a *new* contradiction, surface it — don't silently pick one.

### The cracked-software constraint (drives several decisions above)

A large share of Indian arch-viz agencies run pirated 3ds Max / V-Ray / Corona in
mixed versions (Max 2018–2024, V-Ray 3.x–6.x, Corona 6–11+). Consequences that are
non-negotiable design constraints:

- Agency-side tooling must be **a plain script file with no license check, no
  activation server, and no third-party plugin dependency**. Our `.ms` file, dropped
  into their scripts folder.
- Assume version fragmentation. Detect and branch on `classOf`, never assume a
  property name exists.
- Assume broken installs. Degrade, never crash.

---

## 2. Repository layout

Each subdirectory is an independent project (own toolchain, own tests, own README).
They communicate only through the versioned contracts in `contracts/`.

```
BUILT
  blender_core_processing_p0.1/  Baseline measurement harness (see section 0).
  blender_core_processing_p0.2/  Floor/room detection + cubemap hotspots + viewer.
  docs/             Research + specs. See the reading order below.

ACTIVE — to build
  contracts/        Shared, versioned schemas. THE source of truth across projects.
                      metadata.schema.json    — intake package manifest (agency → server)
                      job-status.md           — job lifecycle states
  exporters/
    maxscript/      3ds Max extractor (MaxScript .ms). Track 1 intake.
  pipeline/         Headless Blender worker (Python + bpy). The core optimizer.
  fixtures/         Real agency test scenes + expected outputs. Gitignored if large.

DEFERRED — do not scaffold (see §0)
  exporters/sketchup/   SketchUp extension (Ruby). Track 2 intake. Phase 2.
  api/                  Ingestion, job queue, status, deploy orchestration. Language TBD.
  dashboard/            Agency web app: upload, status, preview, URL + QR, analytics.
  viewer/               Three.js mobile viewer. Static bundle per project.
  infra/                IaC, deploy scripts, worker images.
  contracts/manifest.schema.json  — delivery manifest (server → viewer), schemaVersion 1.0.0
```

**Rule:** a change to anything in `contracts/` is a breaking change. Bump the
schema version, and update every consumer in the same change.

The pipeline's own boundaries: it takes a path to an unpacked intake package and
writes an optimized GLB plus a processing report. It does not know about HTTP,
queues, databases, or cloud providers.

### `docs/` — what each file is, and how much to trust it

Read in this order. The later technical reports revise the earlier ones; §1 above
is the reconciliation.

| File | What it is | Status |
|---|---|---|
| `ArchVisu_Technical_Feasibility.docx` | Stage-by-stage feasibility, architecture flaws, cost/scale estimates | **Current.** Source of the SketchUp-GLB, conditional-baking, and UV corrections |
| `ArchVisu_3D_Formats_Intake_Strategy.docx` | Format comparison, the cracked-software reality, per-edge-case intake handling | **Current.** Source of the custom-MaxScript decision and the revised tier mix |
| `ArchVisu_Automated_Optimization.docx` | Three-tier UV strategy, per-type decimation ratios, perceptual budgets | **Current.** Source of the §4 invariants |
| `ArchVisu-Requirements_v1.2.md` | Formal PRS/SRS with FR/NFR numbering | **Partly superseded.** Viewer, UX, manifest, and hosting sections still hold; FR-01/02/04/06/19/25 do not — see §1 |
| `Base_Research_Context.md` | The original technical brief | **Historical.** Useful for rationale; its tier percentages and tooling choices are outdated |
| `ArchVisu_India_Market_Research.docx` | Market size, demand signals, competitive landscape | **Current**, and independent of the technical decisions |

All processing-time, file-size, and fidelity numbers across these documents are
estimates. Nothing has been measured on a real agency scene yet.

---

## 3. The pipeline, end to end

```
Track 1 — 3ds Max (client-side, agency's machine)
  MaxScript: read materials → copy bitmaps → sanitize names → normalize units to m
           → record up_axis → recenter if offset → export FBX (geometry+hierarchy)
           → write metadata.json → ZIP

Track 2 — SketchUp (client-side)          [PHASE 2 — design only, build nothing now]
  Ruby ext: (V-Ray present? run "Convert to SketchUp Material" first)
           → native glTF/GLB export (SketchUp 2025+) → metadata.json → ZIP

Server (both tracks converge here)
  unpack → detect intake format → Blender import → classify objects
        → repair geometry → snapshot UVs → per-type decimate → transfer UVs
        → reattach PBR textures (naming convention) → apply glass/emissive presets
        → conditional bake (rare) → texture resize + ORM pack + WebP
        → export GLB → gltfpack (Draco + KTX2)
        → store → PREVIEW to agency → agency approves → deploy → URL + QR
                  └────────── deferred; out of current scope ──────────┘
```

The pipeline's current responsibility ends at "export GLB → gltfpack" plus a
machine-readable processing report (what was decimated, what failed, what was
flagged). Storage, preview, approval, and deploy are downstream and unbuilt.

**Client/server boundary is hard and explicit:** anything requiring 3ds Max or
SketchUp runs on the agency's machine. Nothing else does. There is no server-side
3ds Max.

**Cubemaps are a separate track.** Photoreal 360° panoramas are rendered manually
in Corona by the agency, uploaded separately, and attached to viewpoints in the
manifest. They never enter the Blender pipeline.

---

## 4. Pipeline invariants (`pipeline/`)

These are the tuned defaults. Change them only with a measured reason.

**Order of operations is fixed:** weld → decimate → transfer UVs → reattach
textures → compress → export. Baking, when it happens at all, happens after
decimation. Baking before decimation destroys UVs.

**Weld first, always.** Measured in P0.2: a 3ds Max FBX export arrives with its
meshes exploded into disconnected triangles — one object imported as 115,872
triangles across 115,872 separate islands, 347,616 vertices for a real 58,752.
Until it is welded (`remove_doubles`, 1e-4), collapse decimation has no shared
edges to collapse and every island-based heuristic reads garbage. Welding costs
~1% of triangles. This is not optional and it is not a cleanup step.

**Object classification** — name regex first (catches 60–70%: `wall`, `floor`,
`ceiling`, `tree`, `plant`, `railing`, `handle`, `light`), then geometry heuristics
(bounding-box flatness `min(dims)/max(dims) < 0.05`, poly count, aspect ratio,
loose-island count). Tag every object; write the tag to a custom property for debugging.

**Per-type decimation ratios** (ratio = fraction kept):

| Type | Ratio | Strategy |
|---|---|---|
| Wall / Floor / Ceiling | 0.05–0.15 | Aggressive collapse — planar, 90% reduction is invisible |
| Furniture | 0.3–0.5 | Moderate; preserve silhouette |
| Thin features (railings, mullions, trim) | 0.6–0.8 | **Planar dissolve, not collapse** — these break |
| Vegetation | 0.1–0.2 | Or replace with billboard |
| Decorative / fixtures | 0.4–0.7 | Light |

Stop early if the scene is already under 100K triangles. Prefer meshoptimizer
(via `gltfpack -si -sv`) over Blender's Decimate where thin-feature preservation matters.

**Three-tier UV strategy:**
- Tier A (~80%) — object has UVs: duplicate mesh → decimate → Data Transfer modifier
  (`data_types_loops='UV'`, `TOPOLOGY` with `NEAREST` fallback) → delete duplicate.
- Tier B (~15%) — `len(obj.data.uv_layers) == 0`: Smart UV Project, `angle_limit=66°`,
  `island_margin=0.02`, `correct_aspect=True`. The 66° is load-bearing; defaults over-fragment.
- Tier C (~5%) — needs baking: Lightmap Pack onto a second UV channel.

**Budgets** (phone screens; users notice texture sharpness far more than polycount):
- Scene total: 80K–120K triangles.
- Textures: 1024px walls/floors/hero objects, 512px small/decorative. Never reduce
  normal maps below 1024px.
- **ORM packing is mandatory** — AO→R, Roughness→G, Metallic→B. 5 maps → 3.
- **KTX2/Basis is mandatory, not an optimization.** Without it, 20 materials of
  uncompressed 1024² RGBA ≈ 400MB GPU memory and mid-range phones OOM.
- Target GLB after Draco+KTX2: 5–12MB for a 2BHK. Processing: 3–8 min without baking.

**Material presets, not conversion**, for the hard 2–5%: `type: "glass"` →
alpha 0.3 / roughness 0.05 / transmission 0.9. `type: "emissive"` → emission strength 2.0.
The MaxScript detects and tags these; Blender applies a preset.

---

## 5. Cross-cutting principles

**Never block the agency.** If 3 of 50 materials fail to extract, ship the 47 that
worked, record the 3 as `extraction_status: "failed"` with a `failure_reason`, and
substitute default grey. A degraded result beats a tool that refuses to run —
agencies abandon tools that fail on the first attempt.

**Partial success is a first-class state**, in the export script, the Blender
pipeline, and the job model. Not a linear success/failure sequence.

**Preview before deploy.** Job lifecycle is `queued → processing → preview →
deployed | failed`. Agencies must see the result before their client does. Only
`processing` is in current scope, but the pipeline must emit enough detail in its
report for a preview UI to be built on top without re-running anything.

**Re-uploads need versioning.** Same URL with an updated model vs. a new URL is a
product decision that must be explicit per project, with rollback. Deferred, but
don't design anything that makes reprocessing the same package impossible.

**Defensive input handling is the actual work.** Expect broken texture paths (30%+
of submissions), non-ASCII material names (Hindi/Gujarati), duplicate material
names, network-drive textures, wrong units (40%), 10,000m world offsets, VRayProxy
objects, hidden geometry, unreset transforms. Handle every one explicitly and
record what you did in metadata. `docs/` has the full edge-case table with detection
methods.

---

## 6. Toolchain conventions

**Python — always `uv`.** Never `pip`, `poetry`, `conda`, or a bare `venv`.

```bash
uv init            # new sub-project
uv add <pkg>       # add dependency
uv run <cmd>       # run in project env
uv sync            # reproduce env from lockfile
```

Every Python sub-project gets its own `pyproject.toml` + `uv.lock`, both committed.

### Blender's Python is a real target for `uv` — install into it

Blender ships its own interpreter. It does **not** see the project's `.venv`, but
that is a path problem, not a wall: `uv` can install straight into Blender's
interpreter with `--python`.

```bash
# macOS
uv pip install pydantic --python \
  /Applications/Blender.app/Contents/Resources/5.1/python/bin/python3

# Windows
uv pip install pydantic --python ^
  "C:\Program Files\Blender Foundation\Blender 5.1\5.1\python\bin\python.exe"

# Linux
uv pip install pydantic --python /opt/blender/5.1/python/bin/python3.13
```

Verified on this machine — packages land in
`.../Resources/5.1/python/lib/python3.13/site-packages`, and `--factory-startup`
does **not** hide them (it resets preferences and disables add-ons; it does not
touch `site-packages`).

Two operational notes:

- Match Blender's own Python version. Blender 5.1.2 bundles **3.13.9**, so wheels
  must be cp313-compatible.
- Writing into the app bundle needs permission — admin on Windows `Program Files`,
  and it is wiped by a Blender reinstall. Fine in a container, fragile on a laptop.
- The alternative that leaves the bundle untouched is `PYTHONPATH` plus Blender's
  `--python-use-system-env` flag (Blender ignores `PYTHONPATH` without it). Prefer
  this for local experiments; prefer installing into the interpreter for images.

**So the rule is a preference, not a constraint.** Shared modules imported by both
interpreters should still lean stdlib-only where it costs nothing, because it keeps
`uv run pytest` working with no Blender and no extra install step. But reach for a
real dependency (Pillow for texture packing, pydantic for contract validation,
numpy for grid work) when it earns its place — just install it into **both**
interpreters and record it in the container image.

**Pin Blender to a specific tested version.** Do not auto-update — baking stability
and modifier behavior shift between releases. Pin the packages installed into its
interpreter the same way, in the same place.

**MaxScript / Ruby** have no package manager here. Single-file distribution,
version-stamped in a header comment, matching the `metadata.json` `schema_version`
they emit.

**CLI tools** the pipeline shells out to: `gltfpack` (meshoptimizer) and/or
`gltf-transform`. Pin their versions and document install steps in
`pipeline/README.md` — they're part of the reproducible environment even though uv
doesn't manage them.

**Web** — nothing yet. See §0.

---

## 6a. Containerization is the deployment target

The pipeline is being built to run in a container. That is not a later concern to
retrofit — it decides how the environment is assembled today.

What it means in practice:

- **The image owns the environment.** Pinned Blender, packages installed into
  Blender's interpreter, `gltfpack`/`gltf-transform` binaries, fonts, and any
  OS-level codecs are all baked in and versioned together. "Works on my Mac" is not
  a supported configuration.
- **Installing into Blender's interpreter is the correct move inside an image**,
  and the objection above (mutating an app bundle, lost on reinstall) does not
  apply — the image *is* the install.
- **Keep the CLI seam clean.** Every sub-project stays runnable as
  `uv run <tool> <input> --outdir <path>` with no queue, service, or cloud SDK
  assumed. A container that runs one command over a mounted volume is the unit of
  deployment; wrapping it in a worker comes later (§0).
- **No network at runtime.** Vendored viewer assets, local Draco/KTX2 decoders, and
  packages baked into the image — never fetched at run time. P0.1 and P0.2 already
  hold this line.
- **Everything the run learned goes in `report.json`.** Inside a container the logs
  are ephemeral and stdout is noisy; the machine-readable report is the artifact.
- **Paths come from arguments, never from `__file__`-relative guesses** beyond the
  package root, and never from the user's home directory.

When adding a dependency, the question is not "does it work here" but "what line
does this add to the Dockerfile".

---

## 6b. Blender MCP — use it to check transformations

An MCP server exposing a live Blender session is available in this workspace:

| Tool | Use |
|---|---|
| `get_scene_info`, `get_object_info` | inspect the current scene and a named object |
| `execute_blender_code` | run `bpy` in the live session |
| `get_viewport_screenshot` | see the result |
| `search_polyhaven_assets`, `download_polyhaven_asset` | pull test assets |
| `download_sketchfab_model`, `search_sketchfab_models` | pull test models |

**Use it to verify a geometry or material transformation before writing it into a
pipeline stage.** The alternative — guessing at an operator's parameters, then
waiting three minutes for a full headless run to find out `angle_limit` was in
degrees rather than radians — is the slow way to learn the same fact.

Good uses: confirming an operator name and its arguments against the installed
Blender, checking what an import actually produced, measuring a real object before
tuning a threshold, eyeballing whether a decimation setting wrecked a silhouette.

It does not replace the headless path. Anything that ends up in the pipeline must
still run under `blender --background --factory-startup`, and thresholds tuned
interactively must be pinned by a test.

---

## 7. Working agreements

### Plan first, then execute in small steps

**Every task starts with a plan, not with code.** Before editing anything:

1. **Write the plan.** What is being built, which files change, what the stages are,
   and how it will be verified.
2. **Break it into steps small enough to verify individually.** A step should be
   completable and checkable on its own — "detect floor levels and unit-test the
   peak finder", not "build the detection pipeline".
3. **Execute one step at a time**, confirming each works before starting the next.
4. **Report what actually happened** after each step, including the parts that
   did not work.

Why this is a rule and not a preference: this pipeline's stages depend on each
other in a fixed order, and a wrong assumption early is invisible until much later.
P0.2 found that 3ds Max FBX arrives as disconnected triangles only *after*
decimation silently did nothing — several stages downstream of the actual problem.
Small verified steps surface that on the step that caused it.

It also keeps context manageable. A large task attempted in one pass produces code
nobody has checked, on assumptions nobody has stated.

Corollaries:

- **Surface a blocked or wrong step immediately.** Do not work around it silently
  and continue.
- **State assumptions in the plan**, so a wrong one can be caught before it is
  built on.
- **Measure before tuning.** Every threshold in §4 is a research estimate until a
  real scene says otherwise — check the real value (Blender MCP, §6b) before
  picking a number.

### Everything else

- Don't add a third-party paid dependency to the agency-side path. That decision is
  settled (§1).
- Don't introduce USD/USDZ into v1. Blender's USD importer drops MaterialX materials.
- OBJ is a last-resort fallback intake only, never a primary path.
- Benchmark claims against real fixture scenes before writing them into a spec.
  Processing time and file size numbers in `docs/` are estimates, not measurements.
  `blender_core_processing_p0.1/` can produce real numbers, but so far it has only
  been run on synthetic geometry.
- **Measured so far** (P0.2, one 193K-tri exterior model, Blender 5.1.2, EEVEE):
  full run 1m58s, of which cubemap rendering is 100s and geometry work 16s.
  Object names in a real export are 3ds Max defaults (`Box002`, `Cone001`), so
  §4's name-regex pass caught **0%** — geometry heuristics carried all of it.
  Planar dissolve beat its own target on thin geometry (115,872 → 37,536 tris,
  losslessly). **Texture bytes dominate GLB size**: Draco saved 11% because ~13
  of 16 MB was eight embedded 2K JPEGs, which is direct evidence for the
  "KTX2 is mandatory, not an optimization" line above.
- **Real agency test scenes are the top blocker.** Every tuned number in §4
  (decimation ratios, the 66° angle limit, texture budgets) is a research estimate
  needing validation against actual V-Ray and Corona files. Acquire fixtures early —
  ideally one 3ds Max + V-Ray and one 3ds Max + Corona — and run them through the
  P0.1 harness before committing to any of them.
- This repo is **not currently a git repository.** Initialize before writing code.
