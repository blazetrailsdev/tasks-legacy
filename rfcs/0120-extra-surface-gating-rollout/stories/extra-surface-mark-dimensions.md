---
title: "Widen the extra-surface mark to noCounterpartFiles and allowed"
status: ready
updated: 2026-08-24
rfc: "0120-extra-surface-gating-rollout"
cluster: api-compare
packages: []
deps: []
deps-rfc: []
est-loc: 220
priority: 1
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

The extra-surface ratchet gates two dimensions, `novel` and `total`
(`scripts/api-compare/extra-surface-mark.ts:43-51`), looped over literally at
`:89` and `:114` and merged at `:143-152`. The RFC's enrollment contract needs
two more:

- **`noCounterpartFiles`** — TS files no Rails file maps onto, already computed
  at `scripts/api-compare/extra-surface.ts:1734-1735` and declared at `:732`.
  Their extras are scored with an EMPTY allowed set (`extra-surface.ts:1439-1440`),
  so one new unmapped file dumps its whole public surface into `novel` at once.
  Gating it separately keeps that red distinguishable from real invention.
- **`allowed`** — `pkg.totalAllowlisted`, printed at `extra-surface.ts:1852`.
  Allowed extras are SUBTRACTED from the reported totals, so today a package can
  reach `novel: 0` by tagging rather than deleting and the gate applauds. RFC
  0080 closed with "a ratchet or gate is a follow-up once it is classified";
  classification is done (0 unclassified), so this is that follow-up.

`MeasuredTotals` (`extra-surface-mark.ts:53-57`) currently narrows the report to
`totalNovel` / `totalExtras` and must carry the two new fields.

## Acceptance criteria

- `SurfaceMark` gains `noCounterpartFiles: number` and `allowed: number`;
  `MeasuredTotals` gains the corresponding report fields.
- The dimension list at `extra-surface-mark.ts:89` and `:114` is a single
  exported constant covering all four, used by `exceedances`, `staleMarks` and
  `tightened` — no fourth hand-written loop.
- `tightened()` stays only-shrink for every dimension (`Math.min` per dimension,
  `:146-149` pattern).
- `extra-surface-mark.json` is rewritten through `writeMarks`/`serializeBaseline`
  with arel's measured `noCounterpartFiles` and `allowed` values. arel's `novel`
  and `total` are unchanged. This is a widening of the ROW, not of
  `GATED_PACKAGES`.
- `lint-extra-surface-ratchet.ts`'s summary line and its growth error report name
  the new dimensions.
- `extra-surface-mark.test.ts` covers: growth in each new dimension fails;
  shrink in each is reported stale not failed; `--tighten` narrows each and
  never widens.
- `pnpm parity:api:extra:gate` passes on a clean tree.
