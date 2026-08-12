# Architecture Decision Records

One file per non-obvious decision. Each records what was chosen, **what evidence
drove it**, and what the alternative would have cost.

These exist because most of this pipeline's important choices are
counter-intuitive and were arrived at by measurement, often after the obvious
approach shipped a bug. Without the rationale written down, the obvious approach
gets reintroduced.

## Index

| # | Decision | Status |
|---|---|---|
| [0001](0001-fbx-importer.md) | Prefer `wm.fbx_import` over the legacy FBX importer | Accepted |
| [0002](0002-weld-first.md) | Weld before anything else touches the geometry | Accepted |
| [0003](0003-colorspace-from-wiring.md) | Decide colour space from node wiring, never from filename | Accepted |
| [0004](0004-uv-transfer-recovery-only.md) | UV transfer is recovery-only, never an improvement pass | Accepted |
| [0005](0005-decimation-opt-in.md) | Decimation is opt-in; planar dissolve is repair | Accepted |
| [0006](0006-export-validation-gate.md) | Validate the written GLB, not the scene | Accepted |

## Template

```markdown
# ADR-NNNN — Title

**Status:** Proposed | Accepted | Superseded by ADR-NNNN
**Date:** YYYY-MM-DD

## Context
What forced a decision. Include the measurement if there is one.

## Decision
What was chosen, stated plainly.

## Consequences
What this costs, what it rules out, what to watch for.

## Alternatives considered
What else was on the table and why it lost.
```

## Rules

- **Never edit an accepted ADR to change its decision.** Write a new one and mark
  the old superseded. The record of what was believed and why is the point.
- **Include the number.** "It looked wrong" is not a rationale; "97,920 of 115,872
  faces exceed a 20:1 edge ratio" is.
- **Record failed approaches.** An ADR that says what was tried first and why it
  did not work is worth more than one that only states the answer.
