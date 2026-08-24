---
title: "Report and ratchet @internal counts per package"
status: ready
updated: 2026-08-24
rfc: "0000-extra-surface-gating-rollout"
cluster: api-compare
packages: []
deps: []
deps-rfc: []
est-loc: 200
priority: 2
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`@internal` is the one escape the extra-surface gate cannot see. RFC 0080 drew
the line deliberately: `@noRailsEquivalent` keeps a name COUNTED and reported in
the `Allowed` column, while `@internal` REMOVES it from the compared surface
entirely — `scripts/api-compare/extra-surface.ts:29-30` describes the filter and
`:952` (`if (m.internal === true) return;`) applies it.

That makes `@internal` strictly the cheaper hiding place. The moment the
`allowed` ratchet (`extra-surface-mark-dimensions`) puts pressure on tag counts,
relabelling surface `@internal` becomes the lowest-effort way to stay green, and
nothing reports the move. Every other control in this RFC is a funnel into that
blind spot until this lands.

The extractor already records the flag per declaration
(`scripts/api-compare/extract-ts-api.ts:10`, and `extra-surface.ts:896-897` for
top-level functions taking it from their TS declaration), so this is a
reporting/aggregation change, not new extraction.

## Acceptance criteria

- The per-package report gains an `internal` count — public TS declarations in
  Rails-matched files suppressed by `internal: true` — surfaced in the
  `parity:api:extra` JSON report (the stats-DB consumer shape gains a field; it
  must not lose one) and in a column or summary line of the text table.
- `SurfaceMark` gains an `internal` dimension, ratcheted only-shrink like the
  rest, and arel's mark records its measured value.
- A test asserts that moving a name from `@noRailsEquivalent` to `@internal`
  lowers `allowed` and raises `internal` — i.e. the swap is visible rather than
  silent. This is the whole point of the story.
- `pnpm parity:api:extra:gate` passes on a clean tree.

## Notes

`internal` is a REPORTED dimension, not a moral judgement — most `@internal`
uses are correct (Rails-private helpers, wiring seams). The gate exists so a
SPIKE is visible, not so the count is zero.
