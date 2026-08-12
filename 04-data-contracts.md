# 04 — Data Contracts

Three documents cross a boundary. Each is versioned; **a change to any of them is
a breaking change** — bump the version and update every consumer in the same
change.

| Contract | Direction | Status |
|---|---|---|
| `metadata.json` | agency export tool → pipeline | **Not built.** `contracts/` does not exist yet |
| `report.json` | pipeline → operator, dashboard, tests | Built, schema `0.2` |
| `hotspots.json` | pipeline → viewer | Built, schema `0.2` |

---

## metadata.json — the intake contract

Written by the agency-side export tool, read by the pipeline. **This is the
highest-priority unbuilt artifact**: both the MaxScript extractor and the pipeline
must be written against it, so it should be locked before either is finished.

### Package structure

```
export_v1.zip
├── model.fbx              (or model.glb for the SketchUp path, Phase 2)
├── textures/
│   └── {MaterialName}_{MapType}.{ext}
└── metadata.json
```

### Fields

```jsonc
{
  "schema_version": "1.0.0",
  "source_software": "3dsmax | sketchup",
  "source_version": "2024",
  "renderer": "vray | corona | native",
  "agency_id": "...",
  "project_name": "...",
  "units": "m",              // normalised to metres by the export tool
  "original_units": "in",    // what the scene actually used
  "up_axis": "Z",
  "export_timestamp": "ISO 8601",
  "materials": [
    {
      "name": "WallPaint",             // sanitized
      "original_name": "दीवार पेंट",    // preserved for reference
      "type": "standard | glass | emissive",
      "extraction_status": "ok | partial | failed",
      "failure_reason": null,
      "maps": {
        "baseColor": "textures/WallPaint_BaseColor.jpg",
        "roughness": null,             // null, NOT a generated flat grey
        "normal": null,
        "metallic": null,
        "ao": null
      }
    }
  ]
}
```

### Contested decisions, already resolved

**Emit only the maps that exist; record `null` for the rest.** The older
requirements document mandates always emitting five maps with generated defaults.
`CLAUDE.md` §1 supersedes that: do not ship flat grey PNGs over Indian upload
bandwidth when Blender applies the same defaults for free. Delivery packs to three
maps via ORM anyway.

**`type` drives a preset, not a conversion.** The export tool detects glass and
emissive materials and tags them; the pipeline applies a preset. It does not
attempt to translate a V-Ray node graph.

**Per-material `extraction_status`.** Partial success must be representable at the
material level, or the "ship 47 of 50" behaviour has nowhere to live.

### Open questions

- Does it carry a `bounding_box` for the export gate to check against? Recommended
  — it is the cheapest possible catch for unit and scale bugs.
- How are multi-sub-object materials represented?
- Does the GLB intake path (Phase 2) reuse this file unchanged? It should.

---

## report.json — the processing record

Everything one run learned. Inside a container, logs are ephemeral and stdout is
noisy; **this file is the artifact**.

```jsonc
{
  "schema_version": "0.2",
  "status": "ok | partial | failed",
  "fatal_error": null,

  "input":   { "path": "...", "format": "fbx", "size_bytes": 0 },
  "blender_version": "5.1.2",
  "timings_sec": { "intake": 0.42, "measure": 0.47, "...": 0, "total": 53.6 },

  "geometry": {
    "before": { "objects": 17, "triangles": 191310, "vertices": 94300 },
    "after":  { "objects": 17, "triangles": 191310, "vertices": 94300 },
    "target_ratio": 0.3,
    "actual_keep_ratio": 1.0,
    "actual_reduction_pct": 0.0
  },

  "scene":       { "materials": 15, "images": 7 },
  "floors":      [ /* detected levels, incl. those tagged site/roof */ ],
  "plan_groups": [ /* which storeys share a plan, and how they matched */ ],
  "hotspots":    [ /* mirrors hotspots.json */ ],
  "cubemaps":    { "rendered": 10, "reused": 110, "engine": "...", "size": 1024 },
  "classification": { /* per-class ratios; only when decimation ran */ },
  "objects":     [ /* per-object: class, strategy, before/after, uv tier */ ],

  "outputs": [
    { "name": "model.glb", "kind": "glb", "size_bytes": 0,
      "validation": { "world_size_m": [49.1, 44.74, 88.33], "triangles": 191310,
                      "materials": 15, "images": 7, "passed": true } }
  ],

  "diagnostics_summary": { "info": 11, "warning": 4, "error": 0 },
  "diagnostics": [
    { "severity": "error", "code": "MISSING_TEXTURE",
      "message": "human-readable sentence",
      "obj": "Wall_01", "detail": { "path": "..." } }
  ]
}
```

### Design rules

**`status` is derived, never set directly.** No output → `failed`. Output with at
least one error → `partial`. Output and no errors → `ok`. Warnings never downgrade.

**`code` is stable, `message` is prose.** Software counts and groups by code;
humans read the message. Rewording a message must never break a consumer.

**Requested vs achieved are separate fields.** `target_ratio` and
`actual_keep_ratio` both appear because the gap between them is itself a finding.

**It is always written**, including after a fatal error — outside the top-level
exception handler.

---

## hotspots.json — the viewer contract

```jsonc
{
  "schema_version": "0.2",
  "model": { "glb": "model.glb", "up_axis": "Y", "unit": "m" },
  "eye_height": 1.6,

  "plan_groups": [
    { "id": "g0", "representative_floor": 0, "floors": [0,1,2,3,4,5,6,7,8,9,10,11],
      "match": "iou", "min_iou": 0.9937 }
  ],

  "floors": [ { "index": 0, "label": "Floor 1", "z": 3.3036, "plan_group": "g0" } ],

  "hotspots": [
    {
      "id": "f0_h0",
      "floor_index": 0,
      "plan_group": "g0",
      "position": [43.29, 4.90, -20.69],   // glTF space, Y up, ALREADY converted
      "label": "Room 1",
      "kind": "room | grid",
      "area_m2": 32.0,
      "clearance_m": 1.75,
      "cubemap": { "dir": "cubemaps/g0/h0",
                   "faces": ["px","nx","py","ny","pz","nz"],
                   "ext": "jpg", "size": 1024, "lo": 256 },
      "shares_cubemap_with": null          // or the id that was actually rendered
    }
  ]
}
```

### Design rules

**Positions are already in glTF space.** The viewer performs no axis conversion.
Doing it in two places is how markers end up plausibly wrong.

**The viewer stays dumb.** Every floor gets its own hotspot even when it reuses
another floor's cubemap. Floors sharing a plan simply point at the same directory,
so twelve storeys cost one set of renders and the viewer never has to know.
`shares_cubemap_with` records provenance for debugging only.

**`cubemap.dir` points at a directory, not baked-in files.** This is what lets
Corona-rendered panoramas replace prototype renders with no schema change.

**`kind` distinguishes a detected room from a fallback grid sample**, so the
viewer and the report can tell the user which they are looking at.

**Face order is fixed** and must match three.js `CubeTextureLoader` and the
renderer's orientation table. One ordering, defined once.

---

## Versioning

`schema_version` is semver. Consumers should reject a **major** they do not know
and warn on an unknown minor. Changing any of these files means bumping the
version and updating every consumer in the same change — that is the rule
`CLAUDE.md` §2 states for `contracts/`.
