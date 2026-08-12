# EXPLANATION.md — every file, every function, in plain language

This document explains the entire `blender_core_processing_p0.1/` codebase line by
line. It assumes you can read code but does **not** assume you know Blender, 3D
graphics, or the browser performance APIs. Every piece of jargon is explained the
first time it appears.

`README.md` is the "how do I use it" document. **This** is the "how does it work
and why is it built this way" document.

---

## Table of contents

1. [What this program does, in one paragraph](#1-what-this-program-does)
2. [Jargon dictionary](#2-jargon-dictionary)
3. [The big picture: three worlds that can't see each other](#3-the-big-picture)
4. [`pyproject.toml` — project definition](#4-pyprojecttoml)
5. [`src/p01/report.py` — the shared data model](#5-srcp01reportpy)
6. [`src/p01/blender_locate.py` — finding Blender](#6-srcp01blender_locatepy)
7. [`src/p01/cli.py` — the command you type](#7-srcp01clipy)
8. [`blender/process_model.py` — the actual 3D work](#8-blenderprocess_modelpy)
9. [`fixtures/make_fixtures.py` — generating test models](#9-fixturesmake_fixturespy)
10. [`tests/test_report.py` — the test suite](#10-teststest_reportpy)
11. [`website/` — the measurement viewer](#11-website)
12. [`report.json` — the contract, annotated](#12-reportjson-annotated)
13. [A full run, traced end to end](#13-a-full-run-traced)
14. [What it deliberately does NOT do](#14-what-it-deliberately-does-not-do)

---

## 1. What this program does

You give it a 3D model file. It opens that file in Blender without showing any
windows, writes down everything wrong with the file, throws away 70% of the
model's triangles to make it smaller, saves the result in a web-friendly format,
and then serves a web page that measures how fast that result loads and how
smoothly it runs in a browser.

The reason it exists: the ArchVisu project has a lot of *estimates* about how
fast and how small these models will be, and **zero measurements**. This program
turns estimates into numbers.

---

## 2. Jargon dictionary

Read this once and the rest of the document becomes easy.

### 3D file formats

| Term | What it means |
|---|---|
| **FBX** | A 3D file format from Autodesk. Good at storing shapes and object hierarchy, bad at storing modern materials. This is what 3ds Max exports. |
| **.blend** | Blender's own native file format. Stores everything perfectly, but only Blender can read it. |
| **glTF** | A modern, open 3D format designed specifically for the web. Nicknamed "the JPEG of 3D". |
| **GLB** | glTF packed into a single binary file (model + textures + everything in one). This is what we export, because browsers load it directly. |

### Geometry

| Term | What it means |
|---|---|
| **Mesh** | The actual shape of a 3D object: a list of points in space, plus instructions for which points connect into flat surfaces. |
| **Vertex** (plural **vertices**) | A single point in 3D space. The corners of the shape. |
| **Face / polygon** | A flat surface connecting 3 or more vertices. |
| **Triangle** | A face with exactly 3 corners. **Graphics cards can only draw triangles** — everything else gets chopped into triangles first. This is why we count triangles, not faces. |
| **N-gon** | A face with more than 4 corners. Looks like *one* face in Blender, but the GPU splits it into several triangles. This distinction causes a real bug we avoid — see [`triangle_count`](#triangle_count). |
| **Decimation** | Deliberately deleting triangles to make a model smaller and faster, while trying to keep it looking the same. The core operation of this program. |
| **Normals** | For each surface, an arrow saying "this side is the outside". Determines how light bounces off. Wrong normals make a model look inside-out or weirdly dark. |
| **Degenerate face** | A face with zero area — three corners in a straight line, or two corners in the same spot. Invisible, but it confuses tools and wastes memory. |

### Textures and materials

| Term | What it means |
|---|---|
| **Material** | The recipe for how a surface looks: its colour, shininess, bumpiness. |
| **Texture** | An image file wrapped onto a 3D surface — a photo of bricks wrapped on a wall. |
| **UV / UV map** | Instructions for how to wrap a flat 2D image onto a 3D shape, like the pattern for a sewing project. Called "UV" simply because X, Y, Z were taken, so they used the letters U and V. **No UVs = textures cannot be applied at all.** |
| **PBR** | "Physically Based Rendering" — the modern standard set of textures (colour, roughness, metalness, bumpiness) that all engines agree on. |
| **BSDF** | The maths function describing how light scatters off a surface. **Principled BSDF** is Blender's standard, all-purpose one. glTF understands it directly, which is why we check for it. |

### Blender-specific

| Term | What it means |
|---|---|
| **bpy** | Blender's Python library. `import bpy` only works *inside* Blender. |
| **Headless / background mode** | Running Blender with no windows, driven entirely by a script. |
| **Modifier** | A non-destructive effect stacked on an object — like a filter layer in Photoshop. It changes how the object *looks* without changing the underlying data until you "apply" it. |
| **Depsgraph** | Short for "dependency graph". Blender's calculation of what every object *actually* looks like right now, with all modifiers taken into account. Crucial: the raw mesh data and the depsgraph result can be very different. |
| **Datablock** | Blender's word for any reusable chunk of data — a mesh, a material, an image. |

### Web / browser

| Term | What it means |
|---|---|
| **`<model-viewer>`** | A ready-made HTML tag from Google that displays a GLB file with orbit controls. Saves writing a 3D engine from scratch. |
| **Draco** | A compression algorithm for 3D geometry. Shrinks the *shape* data a lot. **It does not compress textures.** |
| **KTX2 / Basis** | Compression for *textures*. Not used in P0.1 — noted here because it's the missing half of Draco. |
| **rAF** (`requestAnimationFrame`) | A browser function meaning "call me right before the next frame is drawn". Used to measure frame rate. |
| **FPS** | Frames per second. 60 is smooth, 30 is acceptable, below 20 feels broken. |
| **CORS** | A browser security rule that blocks a page from reading files in certain situations. It's why you must serve the site over `http://` rather than opening the file directly. |

---

## 3. The big picture

### Three separate worlds

The single most important structural fact about this codebase:

> **Blender has its own Python, and it cannot see your project's Python packages.**

Blender ships an entire Python interpreter inside itself (version 3.13.9 here).
When you install packages with `uv`, they go into `.venv/` — and Blender's Python
has no idea that folder exists.

This creates three separate worlds:

```
   WORLD 1                    WORLD 2                    WORLD 3
   uv's Python                Blender's Python           Browser
   (3.13.13)                  (3.13.9)                   (JavaScript)

   cli.py                     process_model.py           hud.js
   blender_locate.py          (bpy lives ONLY here)      index.html
   tests/                                                style.css
        │                             │                        │
        └──── launches as ────────────┘                        │
              a subprocess                                     │
                            │                                  │
                            └──→  report.json  ────────────────┘
                                  + .glb files
                                  (files on disk are the ONLY
                                   way these worlds communicate)
```

**Why this matters practically:** if someone adds `import numpy` to
`process_model.py`, it will crash at runtime, and no test will catch it, because
the tests run in World 1 where numpy exists.

### The bridge

One file is shared between World 1 and World 2: [`src/p01/report.py`](src/p01/report.py).
It is written using **only** Python's built-in standard library, so both
interpreters can import it. That's a deliberate constraint, documented in its
docstring. It's how the diagnostics logic gets unit-tested without needing Blender.

### The flow

```
you type:  uv run p01 model.fbx --serve
              │
              ▼
      [World 1] cli.py
        • checks your arguments
        • finds the Blender program on your computer
        • launches Blender as a subprocess
              │
              ▼
      [World 2] process_model.py  (inside Blender)
        1. load the model
        2. inspect it and record every problem
        3. count triangles
        4. delete 70% of them
        5. count again
        6. save two .glb files
        7. write report.json
              │
              ▼
      [World 1] cli.py
        • reads report.json to learn if it worked
        • starts a small web server
              │
              ▼
      [World 3] browser
        • hud.js reads report.json
        • loads the .glb
        • measures load time, size, frame rate
```

---

## 4. `pyproject.toml`

The project's identity card. Three parts matter:

```toml
[project.scripts]
p01 = "p01.cli:main"
```

This is what makes `uv run p01` work. It says: "when someone types `p01`, call the
`main` function inside the `p01.cli` module."

```toml
dependencies = []
```

**Zero runtime dependencies.** Everything uses Python's standard library. This is
intentional — fewer things to break, and it reinforces the discipline that
`report.py` must stay import-free.

```toml
[dependency-groups]
dev = ["pytest>=8.0"]
```

`pytest` is only needed for running tests, not for using the tool, so it's in a
separate development group.

---

## 5. `src/p01/report.py`

**Purpose:** collect everything learned during a run, and turn it into JSON.

This is the shared bridge file. **Stdlib imports only, forever.**

### Constants

```python
SCHEMA_VERSION = "0.1"
SEVERITY_ORDER = ("info", "warning", "error")
STATUS_OK = "ok"
STATUS_PARTIAL = "partial"
STATUS_FAILED = "failed"
```

`SEVERITY_ORDER` is a tuple in increasing order of seriousness — the order is
used later to roll many small findings up into one overall verdict.

The three statuses encode a specific product philosophy from `CLAUDE.md` §5:

- **`ok`** — everything worked
- **`partial`** — we hit errors **but still produced a usable model**
- **`failed`** — we produced nothing

That middle state is the important one. If an agency uploads a scene with 3 broken
textures out of 50, we ship the model with 47 working textures and a list of the
3 problems. We do not refuse to run. A tool that fails on first contact gets
abandoned.

### `class Diagnostic`

One problem found in the model. A dataclass with five fields:

```python
severity: str                  # "info" | "warning" | "error"
code: str                      # "MISSING_TEXTURE" — stable, machine-readable
message: str                   # human-readable sentence
obj: str | None                # which object/material/image it concerns
detail: dict[str, Any]         # extra structured data
```

**Why both `code` and `message`?** The `code` never changes, so software can count
and group by it (`"how many MISSING_TEXTURE errors this month?"`). The `message`
is prose for a human and can be reworded freely.

```python
def __post_init__(self) -> None:
    if self.severity not in SEVERITY_ORDER:
        raise ValueError(...)
```

`__post_init__` runs automatically right after a dataclass is created. This catches
typos like `severity="warnings"` immediately, rather than letting a silently
miscategorised diagnostic flow into the report.

### `class GeometryStats`

Three counters: `objects`, `triangles`, `vertices`. One instance for "before
decimation", one for "after".

### `class Report`

The main container.

#### `__init__`

Sets up empty counters and records `self._t0 = time.perf_counter()`.

> **`perf_counter` vs `time.time()`:** `time.time()` returns wall-clock time, which
> can jump backwards if the system clock syncs mid-run. `perf_counter` is a
> monotonic counter that only ever increases — correct for measuring durations.

#### `add()` / `info()` / `warning()` / `error()`

```python
def add(self, severity, code, message, obj=None, **detail):
    d = Diagnostic(severity=severity, code=code, message=message, obj=obj, detail=detail)
    self.diagnostics.append(d)
    return d
```

`**detail` collects any extra keyword arguments into a dictionary. So calling:

```python
report.error("MISSING_TEXTURE", "not found", obj="Wall", path="/x/y.png", size=0)
```

produces `detail = {"path": "/x/y.png", "size": 0}`. This lets each diagnostic type
carry whatever extra fields make sense, without predefining them all.

`info()`, `warning()`, and `error()` are thin shortcuts around `add()`.

#### `time_phase(name, seconds)`

Records how long a stage took, rounded to 4 decimal places.

#### `add_output(path, draco)`

Records an exported file, reading its size from disk:

```python
"size_bytes": path.stat().st_size if path.exists() else 0
```

The `if path.exists()` guard means a failed export records zero rather than crashing.

#### `counts_by_severity()`

Returns `{"info": 1, "warning": 2, "error": 0}`. Starts from a dict with all three
keys at zero, so the shape is always identical even when nothing went wrong — the
website can rely on the keys existing.

#### `status` (property)

The rollup logic:

```python
if self.fatal_error is not None or not self.outputs:
    return STATUS_FAILED
if self.counts_by_severity()["error"]:
    return STATUS_PARTIAL
return STATUS_OK
```

Read it as: **no output = failed. Output but errors = partial. Output and no
errors = ok.** Note that warnings never downgrade the status — they're
informational.

#### `actual_reduction_pct` (property)

```python
if not self.before.triangles:
    return 0.0
removed = self.before.triangles - self.after.triangles
return round(100.0 * removed / self.before.triangles, 2)
```

The `if not self.before.triangles` guard prevents a division-by-zero crash when
the model failed to load. That's exactly the moment the report matters most, so it
must not crash.

**Why measure this at all when we asked for 70%?** Because decimation cannot always
hit its target — some topology refuses to collapse. The gap between requested and
achieved is a real finding, not noise.

#### `to_dict()`

Assembles the final nested structure. One subtlety:

```python
self.timings.setdefault("total", round(time.perf_counter() - self._t0, 4))
```

`setdefault` only writes if `"total"` isn't already there, so calling `to_dict()`
twice doesn't reset the total.

#### `write(path)`

```python
path.parent.mkdir(parents=True, exist_ok=True)
path.write_text(json.dumps(self.to_dict(), indent=2), encoding="utf-8")
```

`parents=True` creates any missing parent folders; `exist_ok=True` means "don't
complain if it's already there".

#### `summary_lines()`

Builds the short terminal digest. `f"{n:,}"` inserts thousands separators
(`15744` → `15,744`); `/ 1_048_576` converts bytes to megabytes.

---

## 6. `src/p01/blender_locate.py`

**Purpose:** find the Blender program on this computer.

```python
TESTED_VERSION = "5.1.2"
```

The version everything was verified against. `CLAUDE.md` §6 says to pin Blender
because Blender's behaviour shifts between releases. This module doesn't *enforce*
the pin, but it warns loudly on mismatch — a silent difference is worse than a
noisy one.

### `find_blender(explicit=None)`

Checks locations in priority order:

1. `explicit` — the `--blender` command-line flag
2. `$BLENDER_BIN` — an environment variable
3. `shutil.which("blender")` — anywhere on your system `PATH`
4. A hardcoded list of standard install locations (macOS, Linux, Windows)

```python
if path.is_file() and os.access(path, os.X_OK):
    return path
```

Checks both "does this file exist" **and** "is it executable" (`os.X_OK` = the
executable permission bit). A file that exists but can't be run is not a usable
answer.

If nothing is found it raises `BlenderNotFound` with a message telling you exactly
how to fix it. An error that explains its own fix is worth the extra three lines.

### `blender_version(binary)`

Runs `blender --version`, reads the output, returns e.g. `"5.1.2"`.

```python
except (OSError, subprocess.SubprocessError):
    return ""
```

Returns an empty string on any failure rather than raising. Not knowing the version
is a minor inconvenience; crashing over it would be absurd.

---

## 7. `src/p01/cli.py`

**Purpose:** the command you actually type. Validates input, launches Blender,
reports results, optionally serves the website.

### Module constants

```python
PACKAGE_ROOT = Path(__file__).resolve().parents[2]
```

Read this inside-out:
- `__file__` — this file's path (`.../src/p01/cli.py`)
- `.resolve()` — make it absolute, following any symlinks
- `.parents[2]` — go up 2 levels: `p01/` → `src/` → the package root

So the tool works no matter what folder you run it from.

### `build_parser()`

Defines the command-line interface using `argparse`.

```python
p.add_argument("--ratio", type=float, default=0.3,
               help="Fraction of triangles to KEEP. Default 0.3 removes 70%%.")
```

Note `70%%` — in an argparse help string, `%` is a special character, so you write
`%%` to display one literal `%`.

The help text says "**KEEP**" in capitals deliberately: `0.3` meaning "keep 30%"
rather than "remove 30%" is the single most confusable thing in this tool.

Other flags: `--outdir`, `--blender`, `--no-draco`, `--serve`, `--port`, `--no-open`.

### `serve(port, open_browser)`

Starts a small local web server.

```python
handler = functools.partial(
    http.server.SimpleHTTPRequestHandler, directory=str(PACKAGE_ROOT)
)
```

> **`functools.partial`** pre-fills some arguments of a function. `SimpleHTTPRequestHandler`
> needs a `directory`, but the server creates handlers itself and won't pass one.
> `partial` bakes it in ahead of time.

**Why serve `PACKAGE_ROOT` and not `website/`?** Because the page loads the model
from `../3d_model/`. A web server cannot serve files above its root folder. If we
served `website/`, the model would 404. Both folders must sit under one root:

```
PACKAGE_ROOT/           ← server root
├── website/            → http://localhost:8000/website/
└── 3d_model/           → http://localhost:8000/3d_model/   ✓ reachable
```

```python
class ReusableTCPServer(socketserver.TCPServer):
    allow_reuse_address = True
```

Without this, after stopping the server the port stays reserved by the operating
system for a minute or two (the TCP `TIME_WAIT` state), and an immediate restart
fails with "address already in use". This flag says "reuse it anyway".

```python
threading.Timer(0.5, lambda: webbrowser.open(url)).start()
```

Opens the browser after a half-second delay, on a separate thread — the server
hasn't started listening yet at the moment we call this, so opening immediately
would race and show a connection error.

### `main(argv=None)`

The main sequence.

**Step 1 — validate the ratio:**

```python
if not 0.0 < args.ratio <= 1.0:
    ...
    return 2
```

Exit code `2` conventionally means "you used the command wrong", distinct from
`1` = "the work failed". Three distinct exit codes let scripts react correctly.

**Step 2 — validate the input file** exists and has a `.fbx` or `.blend` extension.

**Step 3 — find Blender**, warn on version mismatch.

**Step 4 — build the command:**

```python
cmd = [
    str(blender),
    "--background",         # no window
    "--factory-startup",    # ignore user settings and add-ons
    "--python", str(BLENDER_SCRIPT),
    "--",                   # everything after this is for OUR script
    "--input", str(input_path),
    ...
]
```

Two flags worth understanding:

- **`--background`** runs Blender with no user interface.
- **`--factory-startup`** ignores your personal Blender preferences and installed
  add-ons. Without it, a run on your laptop could behave differently from a run on
  a server. **This makes results reproducible.**

The bare **`--`** is the crucial trick. Blender parses the command line for its
*own* options and would choke on `--input`. Everything after `--` is ignored by
Blender and passed through to our script.

**Step 5 — run it:**

```python
result = subprocess.run(cmd, capture_output=True, text=True)
```

`capture_output=True` grabs the output instead of letting it flood your terminal
(Blender prints hundreds of lines). `text=True` decodes bytes to strings.

**Step 6 — read the outcome from `report.json`**, not from the exit code. The
report is the source of truth.

### `_echo_blender_output(result)`

Blender is extremely noisy. This filters its output down to the handful of lines
our own script printed, by matching the two-space indent and known keywords:

```python
if line.startswith("  ") and any(
    line.strip().startswith(k)
    for k in ("status", "triangles", "objects", "diagnostics", "output", "fatal")
):
```

If the run failed and nothing matched, it dumps the last 3000 characters so you're
never left with no information at all.

### `_report_status(report_path)`

Reads `report.json`, returns the `status` field, or `None` if the file is missing
or corrupt. Never raises.

---

## 8. `blender/process_model.py`

**The heart of the program.** This is the file that runs inside Blender and does
the actual 3D work. 462 lines.

### Constants

```python
MIN_TRIANGLES_TO_DECIMATE = 50
```

Objects with fewer than 50 triangles are left alone. Reducing a 12-triangle door
handle to 4 triangles destroys its shape and saves 8 triangles — a bad trade.

```python
EXTREME_OFFSET_METRES = 1000.0
```

Objects further than 1 km from the world centre get flagged. Real agency files
often have this: someone modelled a building at its real GPS position 10,000 m
from the origin. It breaks camera framing and causes visible jitter, because 3D
maths uses 32-bit floats that lose precision at large magnitudes.

### `parse_args()`

```python
argv = sys.argv[sys.argv.index("--") + 1:] if "--" in sys.argv else []
```

Finds the `--` marker and takes everything after it. This is the receiving end of
the `--` trick from `cli.py`.

### `mesh_objects()`

```python
return [o for o in bpy.context.scene.objects if o.type == "MESH"]
```

A Blender scene contains cameras, lights, empties, and armatures alongside actual
geometry. We only care about meshes.

### <a name="triangle_count"></a>`triangle_count(obj, depsgraph)`

**This function exists because the obvious approach is wrong.**

The obvious approach is `len(obj.data.polygons)`. It's incorrect in two ways:

1. **N-gons.** A face with 6 corners counts as `1` polygon, but the graphics card
   draws it as `4` triangles. A model full of n-gons would report a wildly
   understated triangle count.
2. **Modifiers.** If an object has a Subdivision modifier making it 16× denser,
   the raw mesh data still holds the original low count. You'd be counting the
   "before" while looking at the "after".

The correct approach:

```python
eval_obj = obj.evaluated_get(depsgraph)   # what it looks like WITH modifiers
mesh = eval_obj.to_mesh()                 # bake that into a temporary mesh
mesh.calc_loop_triangles()                # chop all faces into triangles
return len(mesh.loop_triangles)           # count them
```

> **Depsgraph** = Blender's fully-calculated view of the scene. `evaluated_get()`
> asks "what does this object *actually* look like right now".

```python
finally:
    eval_obj.to_mesh_clear()
```

`to_mesh()` allocates temporary memory. `to_mesh_clear()` frees it. The `finally`
block guarantees this runs even if something raises in between — otherwise a large
scene would leak memory on every object.

This is why the reported reduction comes out at a precise `70.01%` rather than a
vague approximation.

### `scene_stats(report)`

Loops every mesh object, summing triangles and vertices. Returns
`(objects, triangles, vertices)`. Called twice — once before decimation, once after.

### `load_model(path, report)`

```python
if suffix == ".blend":
    bpy.ops.wm.open_mainfile(filepath=str(path))
elif suffix == ".fbx":
    bpy.ops.wm.read_factory_settings(use_empty=True)
    _import_fbx(path, report)
```

Two different operations:

- **`.blend`** — *open* the file. It replaces the entire Blender session.
- **`.fbx`** — *import* into the current session. So we first wipe to a completely
  empty scene with `read_factory_settings(use_empty=True)`, otherwise Blender's
  default cube, camera, and light would get exported along with the model.

```python
captured = io.StringIO()
with contextlib.redirect_stdout(captured):
    ...
```

> **`io.StringIO`** is a fake file that lives in memory.
> **`contextlib.redirect_stdout`** temporarily reroutes everything printed into it.

Importer warnings get captured into the report rather than scrolling past in the
terminal where nobody reads them. The result is truncated to 4000 characters,
because a badly broken FBX can emit thousands of lines.

### `_import_fbx(path, report)`

```python
if hasattr(bpy.ops.wm, "fbx_import"):
    try:
        bpy.ops.wm.fbx_import(filepath=str(path))
        return
    except (RuntimeError, TypeError) as exc:
        report.warning("FBX_IMPORTER_FALLBACK", ...)
bpy.ops.import_scene.fbx(filepath=str(path))
```

Blender 5.x ships **two** FBX importers:

- `wm.fbx_import` — new, written in C++, faster
- `import_scene.fbx` — the old Python one

Try the new one; fall back to the old one if it fails. This is not
over-engineering: agency FBX files come from exporters spanning 2018–2024, and the
two importers fail on *different* malformations. Having both roughly doubles the
number of files that load at all. `hasattr` checks the new one exists before
trying, so the code still works on older Blender versions.

### `validate(report)`

The inspection stage. Runs five checks. **None of them raise** — every problem
becomes a recorded diagnostic and the run continues.

```python
if not objs:
    report.error("EMPTY_SCENE", ...)
    return
```

Zero meshes means the file is corrupt, empty, or misidentified. Returns early
because the remaining checks would be meaningless.

#### `_check_textures(report)`

**The most valuable check in the file.** Per the intake research, broken texture
paths affect **30%+ of real agency submissions**.

The problem: when an FBX references `D:\textures\wall.jpg` and that file doesn't
exist, **Blender does not complain**. It creates the image datablock, marks it as
pointing at that path, and renders nothing. No exception, no warning. The model
just quietly looks wrong later.

```python
for img in bpy.data.images:
    if img.source != "FILE":
        continue                    # generated/procedural, no file to check
    if img.packed_file:
        continue                    # embedded inside the .blend, always fine

    raw = img.filepath
    if not raw:
        report.error("MISSING_TEXTURE", "...has no file path")
        continue

    resolved = Path(bpy.path.abspath(raw))
    if not resolved.exists():
        report.error("MISSING_TEXTURE", ...)
    elif tuple(img.size) == (0, 0):
        report.error("UNREADABLE_TEXTURE", ...)
```

> **`bpy.path.abspath`** converts Blender's relative paths (which start with `//`,
> meaning "next to the .blend file") into real absolute paths.

The second check is subtle: `img.size == (0, 0)` means the file **exists** but
Blender couldn't decode it — a corrupt JPEG, or a format it doesn't support. A
different problem needing a different fix, so it gets its own code.

#### `_check_materials(report, objs)`

Three separate checks:

```python
if not obj.material_slots:
    report.warning("NO_MATERIAL", ...)
```
An object with no material exports as plain grey.

```python
if slot.material is None:
    report.warning("EMPTY_MATERIAL_SLOT", ...)
```
A slot exists but nothing is in it — usually a leftover from deleted materials.

```python
has_output = any(n.type == "OUTPUT_MATERIAL" for n in nodes)
has_bsdf = any(n.type == "BSDF_PRINCIPLED" for n in nodes)
if not has_output or not has_bsdf:
    report.warning("MATERIAL_NO_BSDF", ...)
```

**Why specifically Principled BSDF?** Because the glTF exporter understands it
natively and maps it cleanly to glTF's material model. Any other shader setup gets
approximated, sometimes badly. Knowing in advance which materials will be
approximated tells you where to look when the output looks wrong.

#### `_check_uvs(report, objs)`

```python
missing = [o.name for o in objs if not o.data.uv_layers]
```

No UV map means textures **cannot** be applied — there are no instructions for
wrapping the image onto the shape. Only the first 50 names are stored
(`missing[:50]`) so a broken scene doesn't produce a 10 MB report.

P0.1 records this but does not fix it. The fix (`CLAUDE.md` §4) is Smart UV Project
at a 66° angle limit — deliberately out of scope here.

#### `_check_geometry(report, objs)`

Uses `bmesh`, Blender's lower-level mesh library, which exposes per-face and
per-vertex operations the normal API doesn't.

```python
degenerate = sum(1 for f in bm.faces if f.calc_area() < 1e-8)
loose = sum(1 for v in bm.verts if not v.link_faces)
```

- **Degenerate faces** have essentially zero area (`1e-8` = 0.00000001).
- **Loose vertices** belong to no face — invisible points that still consume memory.

```python
finally:
    bm.free()
```

`bmesh` allocates memory outside Python's garbage collector, so it must be freed
explicitly. `finally` guarantees it happens even on error.

#### `_check_placement(report, objs)`

```python
distance = math.sqrt(sum(c * c for c in obj.matrix_world.translation))
```

Pythagoras in 3D: `√(x² + y² + z²)` = straight-line distance from the world centre.

> **`matrix_world`** is the object's final position after all parent transforms are
> applied. `.translation` extracts just the position part.

### `decimate(report, ratio)`

**The core 3D operation.**

```python
modifier = obj.modifiers.new(name="p01_decimate", type="DECIMATE")
modifier.decimate_type = "COLLAPSE"
modifier.ratio = ratio
modifier.use_collapse_triangulate = True
bpy.context.view_layer.objects.active = obj
bpy.ops.object.modifier_apply(modifier=modifier.name)
```

**What COLLAPSE actually does** — the algorithm is *Quadric Error Metrics*
(Garland & Heckbert, 1997):

1. For every vertex, build a small matrix (a "quadric") summarising the flat planes
   of all faces touching it.
2. For every edge, calculate the *error* that merging its two endpoints would cause
   — how far the surface would move from the original.
3. Repeatedly collapse the cheapest edge, merging two vertices into one.
4. Stop when the target triangle count is reached.

The practical consequence is exactly what architecture needs: **flat regions
collapse aggressively, curved detail survives.** A flat wall's interior triangles
cost almost nothing to remove because merging them doesn't move the surface. A
door handle's curves cost a lot, so they're removed last.

**`use_collapse_triangulate = True`** forces triangle output. Without it, collapse
can produce twisted n-gons that render with visible artifacts.

**`objects.active = obj`** — `modifier_apply` is an *operator*, and Blender
operators act on whatever object is currently "active". Setting this is mandatory
or it applies to the wrong object (or errors).

> **Applying a modifier** means baking it permanently into the mesh data. Before
> applying, it's a live preview; after, the triangles are genuinely gone.

The guards:

```python
if tris < MIN_TRIANGLES_TO_DECIMATE:
    skipped += 1
    continue
```

Small objects are skipped entirely.

```python
except Exception as exc:
    failed += 1
    report.error("DECIMATE_FAILED", ...)
```

**Each object is wrapped individually.** One pathological mesh produces one
diagnostic; the other 199 objects still get processed. This is the "never block"
principle in concrete form.

**What this costs:** collapsing vertices interpolates their UV coordinates. At 70%
reduction, curved surfaces develop visible texture stretching and seam artifacts.
P0.1 **does not fix this** — the fix is a Data Transfer modifier that projects the
original UVs back onto the simplified mesh (`CLAUDE.md` §4, Tier A). Leaving it
broken is the entire point: it measures what naive decimation actually costs, so
the fix has something to prove itself against.

### `export_glb(path, draco)`

```python
kwargs = dict(
    filepath=str(path),
    export_format="GLB",
    export_apply=True,
    export_yup=True,
    export_materials="EXPORT",
    export_texcoords=True,
    export_normals=True,
    use_selection=False,
)
```

Each flag:

| Flag | Meaning |
|---|---|
| `export_format="GLB"` | Single binary file, not separate .gltf + textures |
| `export_apply=True` | Bake any remaining modifiers into the exported mesh |
| `export_yup=True` | **Convert coordinate systems — see below** |
| `export_materials="EXPORT"` | Include materials |
| `export_texcoords=True` | Include UV maps |
| `export_normals=True` | Include surface direction data |
| `use_selection=False` | Export everything, not just selected objects |

**`export_yup` is the one that silently ruins things if you get it wrong.**
Different 3D software disagrees about which direction is "up":

- **Blender: Z is up.** (Z-up, like most CAD and architecture software.)
- **glTF: Y is up.** (Y-up, like most game engines and the web.)

This flag applies a −90° rotation about the X axis during export. Without it, every
model appears **lying on its side** in the browser. Nothing errors; it just looks
wrong, and it's a confusing bug to track down because the model is perfect in
Blender.

The Draco settings:

```python
export_draco_mesh_compression_enable=True,
export_draco_mesh_compression_level=6,      # 0-10, higher = smaller but slower
export_draco_position_quantization=14,      # bits of precision for positions
export_draco_normal_quantization=10,
export_draco_texcoord_quantization=12,
```

> **Quantization** = rounding numbers to a fixed grid so they need fewer bits.
> 14 bits gives 2¹⁴ = 16,384 distinct positions per axis — far beyond what the eye
> can detect at architectural scale, but a large saving over 32-bit floats.

Draco is **lossy** (the model is very slightly changed) but imperceptibly so at
these settings.

**Critical limitation:** Draco compresses **geometry only**. Textures are
completely untouched. On our Suzanne test model — which has *no* textures — Draco
achieved 4.2× compression. On a real arch-viz scene where 80% of the bytes are
2K JPEGs, Draco will barely move the number. That result is not a bug; it is
direct evidence for the `CLAUDE.md` §4 claim that **KTX2 texture compression is
mandatory, not an optimization.**

### `main()`

Orchestrates everything.

```python
sys.path.insert(0, str(Path(args.repo_root) / "src"))
from p01.report import Report
```

**This is the bridge between worlds.** Blender's Python doesn't know where our
`src/` folder is, so we add it to the module search path at runtime, *then* import.
The import must come after the `sys.path` line — hence the unusual placement inside
a function rather than at the top of the file.

The sequence, each stage timed:

```python
t = time.perf_counter()
load_model(in_path, report)
report.time_phase("load", time.perf_counter() - t)
```

Then validate → measure before → decimate → measure after → export.

```python
if objs == 0:
    raise RuntimeError("no mesh objects to process")
```

Bails out early — there's nothing to decimate or export.

The Draco export gets its own nested try/except:

```python
try:
    export_glb(draco_path, draco=True)
    report.add_output(draco_path, draco=True)
except Exception as exc:
    report.warning("DRACO_EXPORT_FAILED", ...)
```

If the Draco encoder is missing or fails, we still keep the plain GLB that already
succeeded. Losing the optional extra should never cost you the essential output.

**The outer safety net:**

```python
except Exception as exc:
    report.fatal_error = f"{type(exc).__name__}: {exc}"
    report.error("PIPELINE_FAILED", str(exc))

report.write(report_path)
```

The `report.write()` call sits **outside** the try/except. No matter what explodes,
a valid `report.json` is written explaining what happened. That file is the only
artifact the caller gets, so it must always exist.

```python
return 0 if report.status != "failed" else 1
```

Exit code `0` for ok *and* partial (both produced a usable model), `1` for failed.

---

## 9. `fixtures/make_fixtures.py`

**Purpose:** generate test models so the repo contains no binary files.

Runs inside Blender. Creates two FBX files:

### `build_suzanne(name, texture_path)`

```python
bpy.ops.mesh.primitive_monkey_add(size=2.0, location=(0, 0, 0))
```

> **Suzanne** is Blender's built-in monkey-head test model — the 3D equivalent of
> "Lorem ipsum". It's used because it has curves, creases, and separate parts,
> making it a much better test than a cube.

```python
subsurf = obj.modifiers.new(name="subsurf", type="SUBSURF")
subsurf.levels = 2
bpy.ops.object.modifier_apply(modifier=subsurf.name)
```

> **Subdivision surface** smooths a mesh by splitting each face into four and
> rounding the result. Level 2 = applied twice = roughly 16× more triangles.

This takes Suzanne from ~968 triangles to ~15,744 — enough geometry that a 70%
reduction is actually measurable.

```python
bpy.ops.uv.smart_project(angle_limit=1.15)
```

Generates a UV map. `1.15` is **radians** (≈66°), matching the value specified in
`CLAUDE.md` §4. Blender's Python API takes angles in radians even though the UI
shows degrees.

For the broken fixture:

```python
img = bpy.data.images.new(name="MissingBaseColor", width=8, height=8)
img.filepath = "/nonexistent/path/definitely_not_here_basecolor.png"
img.source = "FILE"
```

Creates an image datablock deliberately pointing at a file that does not exist,
then wires it into the material. The FBX carries that path, and our validator
should catch it on import. This is how we prove the error handling actually works
rather than just claiming it does.

---

## 10. `tests/test_report.py`

15 tests, all running in World 1 with no Blender needed. That's only possible
because `report.py` is dependency-free.

### Helpers

```python
def add_fake_output(report):
    report.outputs.append({"name": "model.glb", ..., "size_bytes": 1234, ...})
```

`status` depends on `outputs` being non-empty. Rather than exporting a real GLB,
the tests append a fake entry directly — fast, and no filesystem involved.

### What's covered

| Test | What it protects |
|---|---|
| `test_rejects_unknown_severity` | Typos like `"warnings"` fail loudly |
| `test_counts_by_severity` | Counting is correct |
| `test_detail_kwargs_are_captured` | `**detail` collects extra fields |
| `test_status_ok_when_output_and_no_errors` | Happy path |
| **`test_status_partial_when_output_despite_errors`** | **An error must not cost us the export** — the single most important behavioural guarantee |
| `test_status_failed_without_output` | No output = failed |
| `test_fatal_error_forces_failed_even_with_output` | A fatal error overrides everything |
| `test_reduction_pct` | 10,000 → 3,000 = 70% |
| **`test_reduction_pct_handles_empty_scene`** | No division-by-zero when nothing loaded |
| `test_reduction_reports_shortfall` | Requested vs achieved are tracked separately |
| `test_round_trips_through_json` | Serialisation preserves everything |
| **`test_writes_even_when_nothing_ran`** | A report is produced even after a fatal crash |
| `test_creates_missing_parent_directory` | `write()` creates folders |
| `test_input_format_derived_from_suffix` | `.FBX` → `"fbx"` (case-insensitive) |
| `test_summary_lines_mention_status_and_outputs` | Terminal digest isn't empty |

The three bolded ones encode the "never block, always report" philosophy. If
someone refactors that away, these tests fail.

---

## 11. `website/`

### `index.html`

The page structure. Key parts:

```html
<script type="module" src="vendor/model-viewer.min.js"></script>
```

Loads the `<model-viewer>` component from a **local** copy, not a CDN.

```html
<model-viewer id="viewer" camera-controls touch-action="none"
              shadow-intensity="0.6" environment-image="neutral">
```

- `camera-controls` — enables drag-to-orbit and scroll-to-zoom
- `touch-action="none"` — stops the browser from scrolling the page when you drag
  on mobile
- `environment-image="neutral"` — uses a built-in soft studio lighting setup, so
  models with no lights still look sensible

Note the element has **no `src`**. The model URL is set by JavaScript later, which
is what allows precise load timing.

The rest is the HUD panel: labelled rows with empty `<b>` elements that JavaScript
fills in.

```html
<link rel="icon" href="data:image/svg+xml,<svg ...fill='%2393c5fd'/></svg>">
```

The favicon is an inline SVG **data URI** — the image is embedded directly in the
HTML rather than being a separate file. Note `%23` — that's a URL-encoded `#`. Left
raw, the browser would read `#` as the start of a URL fragment and silently
truncate the image.

### `style.css`

Standard CSS. Two things worth noting:

```css
:root { --bg: #10131a; --good: #4ade80; --warn: #fbbf24; --bad: #f87171; }
```

CSS custom properties (variables) for the colour scheme. The three status colours
are used by the `grade()` function in JavaScript.

```css
model-viewer { position: fixed; inset: 0; }
```

`inset: 0` is shorthand for `top/right/bottom/left: 0` — makes the viewer fill the
entire window, with the HUD floating on top.

### `vendor/`

Third-party code, committed deliberately:

- `model-viewer.min.js` — Google's 3D viewer component (~955 KB)
- `draco/` — the Draco decoder (`.js` + `.wasm`, ~1 MB)

> **`.wasm`** = WebAssembly, a compiled binary format that runs near-native speed in
> browsers. Draco decoding is maths-heavy, so it's shipped compiled rather than as
> JavaScript.

**Why commit these instead of downloading them?** Two reasons. First, the page
works with no internet. Second and more important: if model-viewer fetched the
Draco decoder from Google's CDN, your measured "load time" would silently include
Google's network latency. You'd be measuring their infrastructure, not your model.

### `hud.js`

426 lines. The measurement logic.

#### Configuration

```javascript
const TARGET = {
  loadMs:    { good: 2000,   warn: 5000 },
  sizeBytes: { good: 10e6,   warn: 20e6 },
  triangles: { good: 120000, warn: 250000 },
  fps:       { good: 50,     warn: 30 },
};
```

These come straight from the project's own targets — the 5-second load requirement,
the 80–120K triangle budget in `CLAUDE.md` §4, and the 10 MB GLB estimate. The HUD
grades against the product's actual goals rather than arbitrary numbers.

#### `grade(value, thresholds, invert)`

```javascript
if (invert) return value >= good ? 'good' : value >= warn ? 'warn' : 'bad';
return value <= good ? 'good' : value <= warn ? 'warn' : 'bad';
```

Returns a CSS class name. `invert` exists because for most metrics lower is better
(load time, size), but for FPS **higher** is better.

#### `loadReport()`

```javascript
const res = await fetch(REPORT_URL, { cache: 'no-store' });
```

`cache: 'no-store'` forces a fresh fetch. Without it, reloading after a new
pipeline run could show stale numbers from the browser cache — a genuinely
confusing bug when you're comparing runs.

#### `renderReport(r)` / `renderDiagnostics(diagnostics)`

Fills in the geometry section and builds the diagnostics list.

```javascript
const before = g.before?.triangles;
```

> **`?.`** is optional chaining. If `g.before` is `undefined`, the whole expression
> returns `undefined` instead of throwing. Makes the HUD robust against a
> partially-written report.

```javascript
const shown = diagnostics.filter((d) => d.severity !== 'info');
```

Only warnings and errors are listed; `info` entries would be noise. Capped at 25
entries.

#### `loadVariant(variant)` — the load measurement

```javascript
const t0 = performance.now();

const done = new Promise((resolve, reject) => {
  viewer.addEventListener('load', resolve, { once: true });
  viewer.addEventListener('error', reject, { once: true });
});

viewer.src = `${url}?t=${Date.now()}`;
await done;
const readyMs = performance.now() - t0;
```

Start the clock, wrap model-viewer's `load` event in a Promise, set `src`, wait.

> **`{ once: true }`** removes the listener automatically after it fires once —
> prevents listeners accumulating each time you switch variants.

**`?t=${Date.now()}`** is a **cache-buster**. Appending a changing query parameter
makes the browser treat it as a new URL and actually re-download. Without it,
switching back and forth would measure the memory cache (≈0 ms), which is useless.

Then the network-level numbers:

```javascript
const entry = performance.getEntriesByType('resource')
  .filter((e) => e.name.includes(variant.name))
  .pop();

const wire = entry.encodedBodySize || entry.transferSize || null;
```

> **Resource Timing API** — the browser records detailed timing for every file it
> downloads. `encodedBodySize` = bytes actually received (after any gzip);
> `transferSize` = that plus HTTP headers, but `0` for cache hits.

`.pop()` takes the *last* matching entry, because switching variants accumulates
multiple entries for similar URLs.

**Why two different load numbers?**

| Number | Measures |
|---|---|
| `entry.duration` | Network transfer only |
| `readyMs` (`src` → `load`) | Network **plus** parsing, Draco decoding, GPU upload |

For a Draco model these diverge sharply — the file is small so the network part is
fast, but decoding costs real CPU time. A network-only metric would make Draco look
strictly better than it is.

#### `renderPageTiming()`

```javascript
const nav = performance.getEntriesByType('navigation')[0];
```

> **Navigation Timing API** — how long the page itself took.
> `domContentLoadedEventEnd` = HTML parsed and scripts run;
> `loadEventEnd` = everything including images finished.

#### `startSampling(maxDurationMs, onFrame, shouldStop)` — the FPS engine

```javascript
function tick(now) {
  const dt = now - last;
  last = now;
  if (dt > 0) frames.push(dt);
  onFrame?.(elapsed, frames);
  const finished = elapsed >= maxDurationMs || shouldStop?.(elapsed, frames);
  if (finished) resolve(frames);
  else requestAnimationFrame(tick);
}
```

Records the gap between consecutive frames. 16.67 ms between frames = 60 FPS.

Two stop conditions: a hard time limit, and a caller-supplied `shouldStop`
predicate. The predicate approach matters — an earlier version tried to `throw`
from inside the callback to stop early, which **does not work**, because a throw
inside a `requestAnimationFrame` callback cannot reject the surrounding Promise.

#### `summarise(frameTimes)`

```javascript
const lowCount = Math.max(1, Math.floor(sorted.length * 0.01));
const worstSlice = sorted.slice(-lowCount);
const lowMean = worstSlice.reduce((a, b) => a + b, 0) / worstSlice.length;
```

Computes average FPS, **1% low** FPS, and worst single frame.

> **1% low** = the average of the worst 1% of frames. This is a standard game
> benchmarking metric, and it matters because *stutter is what people notice*. A
> model can average 60 FPS and still feel broken if it periodically hitches to 10.
> The average hides that; the 1% low exposes it.

#### `wireInteractionFps()` and `runBenchmark()` — why there are two

**This is the most important subtlety in the whole front end.**

`<model-viewer>` is smart: it only re-renders when something changes. Sitting
still, it draws nothing and burns no GPU.

That breaks naive FPS measurement. A `requestAnimationFrame` loop keeps ticking at
your monitor's refresh rate whether or not anything is being drawn. **So an idle
FPS counter reports a confident, meaningless 60.** A model that stutters horribly
under interaction would still show 60 FPS while sitting still.

The two honest answers:

**1. Live FPS** — sample only while the user is actually dragging:

```javascript
viewer.addEventListener('pointerdown', begin);
```

with a 500 ms tail after release to catch the settling frames:

```javascript
(elapsed) => {
  if (pointerUp && tailStart === null) tailStart = elapsed;
  return tailStart !== null && elapsed - tailStart > 500;
}
```

**2. Benchmark** — force continuous rendering for a fixed 10 seconds:

```javascript
const frames = await startSampling(10000, (elapsed) => {
  const theta = startTheta + (elapsed / 10000) * 360;
  viewer.cameraOrbit = `${theta}deg ${phiDeg}deg ${radius}m`;
});
```

Writing `cameraOrbit` every frame guarantees model-viewer redraws every frame. One
full 360° revolution over 10 seconds, so the model is viewed from every angle —
including its worst one. **This is the number worth comparing between models**,
because it's repeatable and doesn't depend on how the user happened to drag.

> **`cameraOrbit`** uses spherical coordinates: `theta` (horizontal angle),
> `phi` (vertical angle), `radius` (distance). The API returns radians; the string
> format wants degrees, hence `* 180 / Math.PI`.

#### `updateEnvironment()` and `detectGpu()`

```javascript
const ext = gl.getExtension('WEBGL_debug_renderer_info');
return ext ? gl.getParameter(ext.UNMASKED_RENDERER_WEBGL) : 'masked by browser';
```

Asks WebGL which graphics card is in use. Many browsers hide this for
fingerprinting-privacy reasons, hence the fallback text. FPS numbers are
meaningless without knowing the hardware, so it's worth showing.

`performance.memory` is Chrome-only, so it's guarded and labelled as such.

#### `init()`

```javascript
await customElements.whenDefined('model-viewer');
const ModelViewerElement = customElements.get('model-viewer');
ModelViewerElement.dracoDecoderLocation = new URL(DRACO_DECODER_PATH, window.location.href).href;
```

**Order is critical.** `whenDefined` waits until the component has finished
registering itself. Only then can we set `dracoDecoderLocation`, and it must be set
**before** any Draco model loads — otherwise model-viewer has already reached for
Google's CDN.

```javascript
await loadVariant(variants.find((v) => !v.draco) ?? variants[0]);
```

Starts on the **uncompressed** variant, because that's the honest baseline. The
Draco number is the improvement you compare against it.

> **`??`** is the nullish coalescing operator — "use the left side unless it's
> `null`/`undefined`". A fallback if no plain variant exists.

---

## 12. `report.json` annotated

The contract between all three worlds. A real run:

```jsonc
{
  "schema_version": "0.1",     // version this format; consumers can check it
  "status": "ok",              // "ok" | "partial" | "failed"
  "fatal_error": null,         // exception string if the run died

  "input": {
    "path": ".../fixtures/suzanne.fbx",
    "format": "fbx",
    "size_bytes": 423116       // source file size, for compression comparison
  },

  "blender_version": "5.1.2",  // which Blender produced this

  "timings_sec": {             // where the time actually went
    "load": 0.0581,
    "validate": 0.0033,
    "decimate": 0.0232,
    "export": 0.0396,
    "total": 0.1247
  },

  "geometry": {
    "before": { "objects": 1, "triangles": 15744, "vertices": 7958 },
    "after":  { "objects": 1, "triangles": 4722,  "vertices": 2437 },
    "requested_ratio": 0.3,           // what we asked Blender for
    "requested_reduction_pct": 70.0,  // the same, expressed as removal
    "actual_reduction_pct": 70.01     // what we actually got
  },

  "scene": { "materials": 1, "images": 0 },

  "outputs": [                 // the website builds its variant buttons from this
    { "name": "suzanne.glb",       "size_bytes": 231016, "draco": false },
    { "name": "suzanne.draco.glb", "size_bytes": 55260,  "draco": true  }
  ],

  "diagnostics_summary": { "info": 0, "warning": 0, "error": 0 },
  "diagnostics": []            // full list, each with severity/code/message/detail
}
```

Note `231016 → 55260` bytes: Draco achieved 4.2× compression here **because
Suzanne has no textures** (`"images": 0`). Expect a far weaker result on a real
textured scene.

---

## 13. A full run, traced

`uv run p01 fixtures/suzanne.fbx --serve`

| # | Where | What happens |
|---|---|---|
| 1 | `cli.py` | `main()` parses args. `ratio=0.3` passes the 0–1 check. |
| 2 | `cli.py` | Input exists, extension is `.fbx`. OK. |
| 3 | `blender_locate.py` | `find_blender()` → `/Applications/Blender.app/...`. Version `5.1.2` matches `TESTED_VERSION`, no warning. |
| 4 | `cli.py` | Builds the command, launches Blender as a subprocess, waits. |
| 5 | `process_model.py` | `parse_args()` reads everything after `--`. |
| 6 | `process_model.py` | `sys.path.insert` + `from p01.report import Report`. The bridge. |
| 7 | `process_model.py` | `load_model()` → empty scene → `wm.fbx_import` succeeds. |
| 8 | `process_model.py` | `validate()` → 5 checks → no problems found. |
| 9 | `process_model.py` | `scene_stats()` → 1 object, **15,744 triangles**, 7,958 vertices. |
| 10 | `process_model.py` | `decimate()` → 15,744 ≥ 50, so Decimate/COLLAPSE at ratio 0.3, applied. |
| 11 | `process_model.py` | `scene_stats()` again → **4,722 triangles** = 70.01% removed. |
| 12 | `process_model.py` | `export_glb()` twice → `suzanne.glb` (231 KB), `suzanne.draco.glb` (55 KB). |
| 13 | `process_model.py` | `report.write()` → `3d_model/report.json`. Prints summary. Exit 0. |
| 14 | `cli.py` | Filters Blender's noisy output down to our 6 summary lines. |
| 15 | `cli.py` | Reads `report.json`, sees `"ok"`. |
| 16 | `cli.py` | `serve()` starts on port 8000, opens the browser after 0.5 s. |
| 17 | `hud.js` | Waits for model-viewer, points it at the local Draco decoder. |
| 18 | `hud.js` | Fetches `report.json`, fills the geometry panel. |
| 19 | `hud.js` | Builds Plain/Draco buttons from `outputs`. |
| 20 | `hud.js` | Loads `suzanne.glb`, times it, reads Resource Timing for the real size. |
| 21 | browser | You drag → live FPS appears. You click Benchmark → 10 s orbit → avg / 1% low / worst. |

---

## 14. What it deliberately does NOT do

P0.1 is a **measuring instrument**, not the product pipeline. `CLAUDE.md` §0 says
explicitly not to grow it into `pipeline/`. Everything below is a conscious
omission, not an oversight:

| Missing | The real pipeline's approach (`CLAUDE.md` §4) | Why omitted here |
|---|---|---|
| Per-object-type decimation | 9 different ratios: walls 0.05–0.15, furniture 0.3–0.5, thin features 0.6–0.8 with *planar dissolve* instead of collapse | A baseline needs one variable, not nine |
| Object classification | Name regex + geometry heuristics to tag each object | Only needed to *choose* per-type ratios |
| UV preservation | Data Transfer modifier projects original UVs onto the decimated mesh | Leaving it broken measures what decimation costs |
| Texture work | Resize to 1024/512, ORM channel packing, KTX2 compression | Out of scope — but this is where the real size wins are |
| Material presets | glass → transmission 0.9; emissive → strength 2.0 | Needs the MaxScript tagging that doesn't exist yet |
| Conditional baking | Bake only procedural materials (~2–5%) | Not needed until real V-Ray scenes arrive |
| `gltfpack` | meshoptimizer for better thin-feature preservation | Blender's built-in Draco is enough for a baseline |
| Any service integration | Queue, storage, API | Deliberate — the CLI seam stays clean |

### The one prediction worth writing down

On the synthetic Suzanne test, Draco compressed the model **4.2×**. This number is
misleading and will not survive contact with a real file.

Suzanne has **zero textures**. Draco compresses geometry only. A real 2BHK
arch-viz scene is dominated by texture bytes — twenty materials of 2048×2048 JPEGs
will completely swamp the mesh data, and Draco will produce a disappointing
single-digit percentage saving.

**When that happens, it is not a bug.** It will be the first hard measurement
supporting the line in `CLAUDE.md` §4 that *"KTX2/Basis is mandatory, not an
optimization"* — and the first real justification for building the texture pipeline
that P0.1 deliberately skips.
