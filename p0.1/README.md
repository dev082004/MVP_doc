# P0.1 — Blender decimation baseline + browser measurement harness

Takes a `.fbx` or `.blend`, loads it in headless Blender, catches whatever is
wrong with it, removes 70% of the triangles, exports GLB (plain and Draco), and
serves a viewer that reports load time, file size, and frame rate.

**This is a measurement instrument, not the product pipeline.** Every performance
number in `docs/` is currently a research estimate. P0.1 exists to replace the
first few of them with measurements, and to give later optimisation work a fixed
baseline to beat.

## Requirements

- Blender **5.1.2** (verified). Set `BLENDER_BIN` if it isn't at the macOS
  default `/Applications/Blender.app/Contents/MacOS/Blender`, or pass `--blender`.
- [uv](https://docs.astral.sh/uv/)

## Usage

```bash
# Generate test fixtures (once)
blender --background --factory-startup --python fixtures/make_fixtures.py

# Process a model and open the viewer
uv run p01 fixtures/suzanne.fbx --serve

# Options
uv run p01 model.fbx --ratio 0.5      # keep 50% instead of 30%
uv run p01 model.fbx --no-draco       # skip the compressed variant
uv run p01 model.fbx --serve --port 9000
```

Outputs land in `3d_model/`:

```
3d_model/
├── <name>.glb          uncompressed baseline
├── <name>.draco.glb    Draco-compressed variant
└── report.json         diagnostics, timings, before/after geometry
```

Exit codes: `0` success, `1` processing failed, `2` bad arguments.

### The viewer must be served over HTTP

Opening `website/index.html` as a `file://` URL will not work — the browser
blocks fetching `report.json` and the GLB. Use `--serve`, or:

```bash
python3 -m http.server 8000      # from THIS directory, not website/
# then open http://localhost:8000/website/
```

The server root has to be this directory because the page loads the model from
`../3d_model/`.

## What the HUD measures

| Readout | Source | Why it's separate |
|---|---|---|
| Size (on wire) | `PerformanceResourceTiming.encodedBodySize` | What actually crossed the network, gzip included |
| Size (on disk) | `report.json` | Cross-check; catches server-side compression |
| Network fetch | Resource timing `duration` | Transfer only |
| Fetch → renderable | `src` set → `load` event | Transfer **plus** parse, decode, and GPU upload |
| Live FPS | rAF, sampled only while dragging | See below |
| Benchmark | rAF during a forced 10s orbit | The comparable number |

### Why there are two FPS numbers

`<model-viewer>` renders on change, not continuously. A `requestAnimationFrame`
counter left running on an idle page measures the display's refresh rate and will
happily report a flat 60 fps for a model that actually stutters.

So live FPS is sampled **only while a pointer is down**, and the benchmark button
forces continuous rendering by orbiting the camera every frame for 10 seconds,
reporting average, 1% low, and worst frame time. The 1% low is what stutter
actually feels like.

Colour thresholds come from the product targets: load <5s, ≥30 fps,
≤120k triangles, ≤10 MB.

## Deliberate deviations from CLAUDE.md §4

P0.1 is intentionally cruder than the designed pipeline. Do not mistake it for it:

| CLAUDE.md §4 | P0.1 | Why |
|---|---|---|
| Per-object-type decimation ratios | One uniform ratio for everything | A baseline needs one variable, not nine |
| UV preservation via Data Transfer | Plain Decimate; UV loss recorded, not fixed | Measures what naive decimation costs |
| ORM packing, texture resize, KTX2 | None | Out of scope |
| `gltfpack` post-process | Blender's built-in Draco instead | No extra binary needed yet |

Also not here: per-object classification, glass/emissive presets, conditional
baking, and any queue/API/storage integration. P0.1 is a standalone CLI —
`CLAUDE.md` §0 keeps that seam clean.

## Layout

```
src/p01/report.py         PURE STDLIB — shared with Blender's interpreter
src/p01/cli.py            arg parsing, Blender discovery, subprocess, --serve
src/p01/blender_locate.py
blender/process_model.py  runs INSIDE Blender (bpy + stdlib only)
website/                  model-viewer + HUD, fully vendored (works offline)
fixtures/make_fixtures.py
tests/                    uv run pytest — no Blender needed
```

`report.py` is deliberately dependency-free because it is imported by two
different interpreters: the uv environment and Blender's bundled Python, which
cannot see uv-installed packages.

## Diagnostics the loader catches

`EMPTY_SCENE`, `MISSING_TEXTURE`, `UNREADABLE_TEXTURE`, `EMPTY_MATERIAL_SLOT`,
`NO_MATERIAL`, `MATERIAL_NO_BSDF`, `MATERIAL_NO_NODES`, `NO_UV`,
`DEGENERATE_FACES`, `LOOSE_VERTICES`, `EXTREME_OFFSET`, `DECIMATE_FAILED`,
`FBX_IMPORTER_FALLBACK`, `DRACO_EXPORT_FAILED`.

None of them abort the run. A model with problems still gets exported, with the
problems recorded — partial success is a first-class state (`CLAUDE.md` §5).

## Tests

```bash
uv run pytest
```

Covers the diagnostic/status rollup and reduction maths. The Blender stages are
verified end-to-end against the generated fixtures:

- `suzanne.fbx` → `status: ok`, exactly 70.01% removed
- `broken_texture.fbx` → `status: partial`, `MISSING_TEXTURE` recorded, GLB still
  exported
- a corrupt file → `status: failed`, exit 1, `report.json` still written and readable
