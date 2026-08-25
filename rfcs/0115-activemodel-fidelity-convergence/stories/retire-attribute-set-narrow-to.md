---
title: "retire-attribute-set-narrow-to"
status: ready
updated: 2026-08-25
rfc: "0115-activemodel-fidelity-convergence"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`packages/activemodel/src/attribute-set.ts` still carries `narrowTo`, a trails
invention with no counterpart in
`vendor/rails/activemodel/lib/active_model/attribute_set.rb` (91 code lines,
none of them a narrowing writer). It survives `parity:api:extra` only because
it is tagged `@internal`, so it is unmeasured debt rather than allowed surface.

Its one production caller is `narrowToProjectedColumns`
(`packages/activerecord/src/inheritance.ts:550`), reached from
`Base.instantiate` (`packages/activerecord/src/base.ts:2819`) after every row
is written attribute-by-attribute with `writeFromDatabase`. Rails never does
this: `instantiate` hands the row to `attributes_builder.build_from_database`
(`activerecord/lib/active_record/persistence.rb:82`), and
`LazyAttributeSet#keys` / `#key?`
(`activemodel/lib/active_model/attribute_set/builder.rb:32-39`) already report
only the projected columns because the unprojected ones were never in `values`.
The narrowing pass exists to retrofit that onto a set that was eagerly filled
from the full schema.

RFC 0115's `retire-attribute-set-map-adapter-surface` (PR #TBD) retired the Map
facade, the freeze/snapshot/clone sidecars and `overrideFromDatabase`, taking
`attribute-set.ts` to 0 novel / 0 moved; `narrowTo` was left because removing it
means routing `instantiate` through `Builder#build_from_database`, which is its
own PR.

## Acceptance criteria

- `AttributeSet#narrowTo` is deleted, along with
  `narrowToProjectedColumns` in `inheritance.ts`.
- `Base.instantiate` builds the attribute set through
  `AttributeSet::Builder#build_from_database(row, columnTypes)`
  (`packages/activemodel/src/attribute-set/builder.ts`), so a projected-subset
  SELECT reports the loaded columns from `keys`/`isKey` with no post-pass.
- The STI path in `inheritance.ts` is converged with it (same builder entry).
- `pnpm parity:api:extra --package activemodel` keeps `attribute-set.ts` off the
  divergence table; parity deltas non-negative for activemodel and activerecord.

## Verification

```bash
pnpm vitest run packages/activerecord/src/inheritance.test.ts packages/activerecord/src/base.trails.test.ts packages/activerecord/src/attribute-methods
```
