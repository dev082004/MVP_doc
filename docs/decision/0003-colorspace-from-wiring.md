# ADR-0003 — Decide colour space from node wiring, never from filename

**Status:** Accepted
**Date:** 2026-08-08

## Context

Blender's FBX importer marks **every** texture as sRGB. That is wrong for data
maps: a roughness or metalness map read through an sRGB transfer curve gives the
wrong surface response everywhere it appears.

So they need correcting to `Non-Color`. The obvious way is to read the filename —
`*_Roughness`, `*_Normal`, `*_METALNESS` are unambiguous, and this is what most
pipelines do.

**It is worse than the disease.** Measured on the intake asset: five of its seven
textures are named `*_AmbientOcclusion`, `*_Displacement` and `*_METALNESS` — and
**every one of them is wired into Base Color.**

The artist dragged whatever map looked right into the diffuse slot. That is
precisely the drag-and-drop library-material behaviour the intake research
describes as the dominant workflow. An earlier implementation trusted the name and
**corrupted the diffuse of most of the building**.

The filename describes what the map *was*. Only the wiring describes what it is
*being used as*.

## Decision

Decide from how the texture node is connected:

1. If it feeds a node that consumes data by definition — Normal Map, Bump,
   Displacement — it is **data**, whatever the socket is called.
2. Otherwise, if it feeds a shader node, the **socket name** decides: Base Color,
   Emission Color, Tint and similar are colour; everything else is data.
3. If nothing conclusive is wired — it passes through a Mix, a Math node, a group,
   or nothing at all — **only then** fall back to the filename.

Record which path decided (`decided_by: "wiring" | "name"`) so the choice is
auditable.

### Three details that carry weight

**Data-consumer nodes are checked first.** A Normal Map node's input socket is
*also* called "Color". Socket-name logic alone gets it exactly backwards.

**The Alpha output carries no verdict.** Alpha is a separate channel and says
nothing about the colour space of the RGB, so links leaving it are skipped.

**Colour wins a disagreement.** A texture feeding both Base Color and Roughness
stays sRGB. A diffuse map read as data is glaring; a roughness map read as colour
is subtle. Fail toward the less visible error.

**Only touch sRGB ↔ Non-Color.** Anything the importer set deliberately (Raw,
Linear Rec.709) is left alone.

## Consequences

- The fallback regex must be boundary-anchored. Unanchored, `ao` matches half the
  words in English and `orm` matches "platform", "normal" and "transform".
- Textures that reach a shader through a node group are inconclusive by design,
  and fall back to the name. Acceptable — arch-viz library materials rarely use
  groups.
- The decision is only as good as the wiring. A material with a data map genuinely
  plugged into Base Color will keep it as sRGB — **which is correct**, because
  that is how it is being used and how Blender renders it.

## Alternatives considered

**Filename only.** Rejected on measurement — it corrupted 5 of 7 textures on the
first real asset tested.

**Ask the export tool to tag map types in `metadata.json`.** Genuinely better, and
should happen. Rejected as the *sole* mechanism because it only helps files that
came through our own MaxScript, and the pipeline must handle files that did not.

**Leave everything sRGB.** Rejected — it is wrong for any correctly-authored
material, and correctly-authored materials are the case we want to get right as
the asset population improves.

## Notes

This ADR and [ADR-0001](0001-fbx-importer.md) share a lesson worth stating
directly: **in this domain, names are unreliable and structure is reliable.**
Object names are 3ds Max defaults, material names are decorative, texture names
describe the download rather than the use. Every heuristic that depends on a
string should be treated as a last resort.
