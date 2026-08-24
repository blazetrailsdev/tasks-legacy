---
title: "Burn down activerecord extra surface from 399 novel to enrollment"
status: draft
updated: 2026-08-24
rfc: "0120-extra-surface-gating-rollout"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 200
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

PR #6997 froze `activerecord`'s extra surface at the measured high-water mark so it
cannot grow. It did not reduce it. This story is the reduction.

Measured 2026-08-24 on `main` (152b2ebe9), stable across two consecutive runs on
a clean tree with rebuilt manifests:

| package      | files | novel | moved | total | allowed | noCntrp |
| ------------ | ----- | ----- | ----- | ----- | ------- | ------- |
| activerecord | 232   | 399   | 1025  | 1424  | 108     | 383     |

For scale, `arel` finished its RFC 0117 burndown at `novel: 0, total: 63`.
`activerecord` is the largest population in the repo.

The run also reports:

- `@noRailsEquivalent` tags: 215 total, 63 matched (+18 allowed by a tagged
  interface declaration).
- Permanence claims: **204 PERMANENT, 11 CONVERGEABLE, 0 unclassified.**
- Excluded by kind: 741 novel `interface` declaration names and members —
  type-only shapes Ruby leaves to duck typing, already out of the scored
  population.

That 204 PERMANENT against 11 CONVERGEABLE is worth reading before starting: it
is the ratio this RFC's §`@noRailsEquivalent` section warns about, where the
cheap path to `novel: 0` is tagging rather than deleting. A burndown that moves
`novel` down while `allowed` moves up has not converged anything.

### Sequencing

`noCntrp: 383` of the 1424 total comes from TS files no Rails file maps onto.
Per this RFC's contract, `noCounterpartFiles` is the leading indicator and is
largely fixed in `scripts/parity/conventions.ts`
(`PATH_SEGMENT_ALIASES` / `RUBY_FILE_TS_OVERRIDES`) rather than in trails source.
Doing the file mapping first is likely to move a large slice of `novel` with no
source change, and doing it last means burning down names that were never
invented.

RFC 0119 (connection-adapter fidelity) burns down part of this population as a
side effect — roughly 25 of its ~106 open stories delete trails-only surface in
`connection-adapters/**`. Coordinate rather than double-count.

## Acceptance criteria

- `activerecord` reaches `novel: 0` and `noCounterpartFiles: 0`, satisfying this
  RFC's enrollment contract, and its mark is tightened with
  `pnpm parity:api:extra:tighten` as it falls — never reseeded, never raised.
- The `allowed` count does not rise over the burndown. A story that would need a
  new `@noRailsEquivalent` tag to reach zero is **blocked, not tagged** — the
  discipline RFC 0117 used ("arel may finish with at most 8 tags").
- Every surviving `@noRailsEquivalent` in `packages/activerecord/src` carries a
  reviewed permanence claim, and each `CONVERGEABLE` names a filed story id per
  `convergeable-tag-story-id`.
- Depends on the tier question in
  `reconcile-activerecord-non-zero-gate-enrollment` being settled first: if
  `activerecord` comes out of `GATED_PACKAGES`, this burndown is what puts it
  back.

## Notes

Expect this to be many PRs. `est-loc` is for the first slice (the
conventions.ts file-mapping pass), not the whole population.
