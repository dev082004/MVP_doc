# 01 — Scope and Boundaries

## What the pipeline is

A **standalone command-line program**. It takes a path to an unpacked intake
package and writes an optimized GLB plus a machine-readable processing report.

```
pipeline <intake-package-path> --outdir <path>
   ↓
<outdir>/model.glb          optimized, web-ready
<outdir>/report.json        everything the run learned
```

It knows nothing about HTTP, queues, databases, cloud providers, or users.

## What it is responsible for

| Responsibility | Detail |
|---|---|
| **Intake** | Import FBX/GLB, repair geometry, relink textures, normalise materials |
| **Analysis** | Detect storeys and rooms from geometry alone; group repeated floor plans |
| **Capture** | Render a cubemap at each capture point (prototype only — see below) |
| **Optimize** | Reduce triangles and texture weight to a web budget |
| **Export** | Write GLB, then validate the written file |
| **Report** | Emit a complete, machine-readable account of what happened |

## Where it stops

The pipeline's responsibility **ends at files on disk**. It does not upload,
deploy, notify, or track jobs.

```
agency machine        │  PIPELINE (this scope)          │  downstream (not this)
─────────────────────────────────────────────────────────────────────────────
3ds Max + MaxScript   │  unpack → import → repair       │  store
export tool           │  → analyse → render → optimize  │  preview to agency
                      │  → export GLB → report.json     │  approve → deploy
                      │                                 │  URL + QR + analytics
```

## Deliberate exclusions from these documents

### Containerization

Not documented here. The pipeline is *built* to run in a container — that decides
how the environment is assembled — but that is an environment concern, recorded in
`CLAUDE.md` §6a. What matters to this scope is only the consequence:

- the CLI seam stays clean (no queue, service or cloud SDK assumed)
- **no network at runtime** — vendored assets, local decoders, packages baked in
- paths come from arguments, never from `__file__` guesses or the user's home
- `report.json` is the artifact, because container logs are ephemeral

### Job queue and orchestration

Not designed here. `CLAUDE.md` §0 records the backend/queue language as
**undecided**. Designing a queue now would force that choice by implication.

The pipeline only has to remain *wrappable*: a single command, deterministic
outputs, a status readable from `report.json`, and no assumption that anything is
calling it.

### Cloud platform

Not chosen. `CLAUDE.md` §0 records storage as undecided (AWS S3 vs Cloudflare R2)
and the research documents disagree. The requirement on this scope is narrow:
**reads and writes go behind a storage interface, and no provider SDK appears in
pipeline logic.**

### Viewer and dashboard

Out of scope. `viewer/` is deferred per `CLAUDE.md` §0. The prototypes ship a
`website/` for verification only — it exists to prove the GLB is correct, not to
be the product.

## Hard constraints on this scope

**Client/server boundary is absolute.** Anything requiring 3ds Max or SketchUp
runs on the agency's machine. There is no server-side 3ds Max. The pipeline never
opens a `.max` file.

**Track 1 (3ds Max → FBX) is the only intake path in current scope.** SketchUp
(Track 2, GLB direct) is Phase 2. The data contracts must be able to represent a
GLB intake so they do not change when it arrives, but no SketchUp code is written.

**Partial success is a first-class outcome.** A run that hits errors and still
produces a usable model must say so and ship the model. Refusing to run is worse
than degrading. See [05 — Error model](05-error-model.md).

**Cubemaps are a separate production track.** Photoreal panoramas are rendered by
the agency in Corona and attached to viewpoints by the manifest — they do not
enter the Blender pipeline. The prototype renders them in Blender because a
prototype needs imagery; the contract points at a *directory of face images*, so
Corona output replaces it with no schema change.

## Current build status

| Stage | Status |
|---|---|
| Intake, repair, texture relink, material fix | Built and measured |
| Floor / room / plan detection | Built and measured |
| Cubemap render | Built (prototype-only by design) |
| Decimation | Built, **opt-in**, currently off by default |
| Texture optimization (resize, ORM, KTX2) | **Not built** — the largest remaining gap |
| GLB export + validation gate | Built |
| `contracts/` schemas | **Not built** |
