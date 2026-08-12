# 05 — Error Model

## The principle

**Never block the agency.** If 3 of 50 materials fail to extract, ship the 47 that
worked, record the 3, and substitute default grey. A degraded result beats a tool
that refuses to run — agencies abandon tools that fail on first contact.

This is not a nicety. Real submissions arrive with broken texture paths, non-ASCII
material names, duplicate names, wrong units, 10 km world offsets, proxy objects,
hidden geometry and unreset transforms. A pipeline that treats any of those as
fatal processes almost nothing.

## Three outcomes

| Status | Meaning | Exit |
|---|---|---|
| `ok` | Output produced, no errors | 0 |
| `partial` | **Output produced despite errors** | 0 |
| `failed` | No usable output | 1 |

Bad arguments exit `2`, distinct from processing failure, so a wrapper can tell
"you called it wrong" from "the model was broken".

`partial` is the state that matters and the one most pipelines omit.

## Derivation

```
fatal_error set, or no outputs  →  failed
any error-severity diagnostic   →  partial
otherwise                       →  ok
```

Status is **derived, never assigned**. Warnings never downgrade it.

## Diagnostic shape

```jsonc
{
  "severity": "info | warning | error",
  "code": "MISSING_TEXTURE",      // stable, machine-readable
  "message": "Texture 'x' ...",   // prose, freely rewordable
  "obj": "Wall_01",               // object, material or image
  "detail": { "path": "..." }     // structured extras, per code
}
```

Codes are declared as constants, so a typo is an import error rather than a
silently unmatched string in a test.

### Severity meaning

| Severity | Meaning | Effect |
|---|---|---|
| `info` | Worth knowing; nothing is wrong | none |
| `warning` | Degraded but handled; a human may want to look | none on status |
| `error` | Something is genuinely wrong with the output | forces `partial` |

The line that matters: **an error means the delivered model is wrong in a way the
agency should know about**, not that the pipeline failed.

## Code registry

### Intake

| Code | Severity | Meaning |
|---|---|---|
| `EMPTY_SCENE` | error | File imported but contains no meshes |
| `FBX_IMPORTER_FALLBACK` | warning | Preferred importer failed; legacy used — expect fabricated material values |
| `MISSING_TEXTURE` | error | Referenced texture could not be found |
| `UNREADABLE_TEXTURE` | error | File exists but could not be decoded |
| `TEXTURE_RESOLVED_BY_FALLBACK` | warning | Relinked by stem match, not the recorded path |
| `TEXTURE_COLORSPACE_CORRECTED` | info | Colour space changed; records whether wiring or name decided |
| `MATERIAL_IMPORT_ARTIFACT` | warning | Importer defaults detected and corrected |
| `NO_MATERIAL` | warning | Mesh had none; default grey substituted |
| `EMPTY_MATERIAL_SLOT` | warning | Slot exists but is empty |
| `MATERIAL_NO_BSDF` | info | glTF export will approximate this material |
| `NO_UV` | warning | No UV layer; unwrapped procedurally |
| `EXTREME_OFFSET` | warning | Object far from world origin; float precision suffers |
| `LOOSE_VERTICES` | info | Weld/cleanup results |
| `DEGENERATE_FACES` | info | Zero-area faces removed |

### Analysis

| Code | Severity | Meaning |
|---|---|---|
| `NO_FLOOR_LEVELS_DETECTED` | warning | No storeys found; room detection and capture skipped |
| `NO_ENCLOSED_SPACE` | warning | Plan encloses nothing at eye height |
| `NO_ROOMS_DETECTED_USING_GRID` | warning | No room-sized spaces; fallback viewpoints placed |
| `HOTSPOT_OBSTRUCTED` | warning | Capture point too close to geometry; render may be mostly wall |

### Capture and optimize

| Code | Severity | Meaning |
|---|---|---|
| `AUTO_LIGHTING_ADDED` | info | Scene had no lights; sky and sun added |
| `CUBEMAP_RENDER_FAILED` | error | A face failed; that point has no panorama |
| `TRIANGLE_BUDGET_MISSED` | warning | Solver floored out; records why |
| `DECIMATE_FAILED` | error | Object kept at full resolution |
| `UV_TRANSFER_FAILED` | error | UVs lost and unrecoverable |
| `UV_DISTORTED` | error | UV area moved implausibly; textures on this object will be wrong |

### Export gate

| Code | Severity | Meaning |
|---|---|---|
| `GLB_EXPORT_FAILED` | error | Export raised |
| `GLB_INVALID` | error | Written file could not be parsed back |
| `SCALE_MISMATCH` | error | Exported bbox disagrees with the Blender scene |
| `MATERIAL_METALLIC_IMPLICIT` | error | `metallicFactor` omitted — glTF reads it as 1.0 |
| `TEXTURE_UNRESOLVED` | error | Image references neither embedded data nor a URI |
| `MESH_WITHOUT_UV` | error | Primitive has no `TEXCOORD_0`; renders untextured |
| `TRIANGLE_COUNT_MISMATCH` | error | Written geometry disagrees with the scene |
| `GLB_OVER_BUDGET` | warning | File exceeds the advisory size ceiling |
| `DRACO_EXPORT_FAILED` | error | Compressed variant failed; plain still shipped |

## Structural rules

**Wrap per object, not per stage.** One pathological mesh produces one diagnostic;
the other 199 objects still process. A stage-level try/except loses the other 199.

**The report is written outside the top-level handler.** Whatever explodes, a
valid `report.json` exists explaining what happened. It is the only artifact the
caller is guaranteed.

**Cap unbounded detail.** A broken file can emit thousands of lines or name
thousands of objects. Truncate lists and captured output — a 10 MB report helps
nobody.

**Optional work never costs required work.** A failed Draco variant must not lose
the plain GLB that already succeeded. Nest its handler.

## Why the export gate exists

Two bugs shipped that no triangle count could have caught:

- every material silently defaulting to fully metallic
- meshes exporting with no UVs after a stage was made optional

Both were structural properties of the *written file*, with perfect triangle
counts and no exception anywhere. The only honest check is to read the file back
and assert on it.

**A model that passes the gate but still looks wrong is a gap in the gate.**
Tighten the checks rather than falling back on manual visual QA.
