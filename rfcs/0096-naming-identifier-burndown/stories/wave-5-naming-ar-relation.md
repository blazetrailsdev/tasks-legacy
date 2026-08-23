---
title: "Wave 5: burn down the AR-closure naming rows in activerecord relation, scoping and statement-cache"
status: done
updated: 2026-08-23
rfc: "0096-naming-identifier-burndown"
cluster: api-compare
packages: ["activerecord"]
deps: []
deps-rfc: []
est-loc: 80
priority: 14
pr: 6917
claim: "2026-08-23T14:04:32Z"
assignee: "wave-5-naming-activesupport"
blocked-by: null
closed-reason: null
---

## Context

RFC 0096's closing story `naming-gate-flip` is blocked on the AR require-closure
reaching **zero convergeable `naming` rows** (`burndown` +
`module-mixin-receiver`). Waves 1-4 drained the population it was scoped
against; a fresh reading of
`scripts/api-compare/output/call-arg-mismatches.json` (artifact of 2026-08-21,
rendered by `pnpm parity:api:calls:args:report`) shows **107 convergeable rows
still standing inside the closure**, across 67 files. This wave-5 band splits
those 107 into six non-overlapping file sets so the flip has a defined finish
line again.

Rules are unchanged from the RFC's `## Design`:

- **Rename to the Rails identifier, not to a better one.** If Rails writes `o`,
  the TS local is `o`, camelCased per `docs/ruby-ts-conventions.md`.
- **Body-local only.** No behavior change, no public surface change.
- **A row that is really an a1 (argument order) or a3 (invented helper /
  conversion) finding is NOT renamed away.** File it against the RFC that owns
  the file and leave the row. Several rows below look like that shape — e.g.
  `activesupport/notifications.ts#instrument` passes a different argument list,
  not a differently-named one — so read each pair against the vendored Ruby
  before renaming.
- `module-mixin-receiver` rows converge by rewiring to the `this`-typed mixin
  idiom (CLAUDE.md, Module mixins), not by renaming the parameter.

## Rows in this slot

15 rows across 9 files. **File set:** `activerecord/relation*`, `activerecord/scoping/`, `activerecord/statement-cache.ts` only.

- `activerecord/relation.ts` — 4
  - `update_all`: ruby `ref:uniq` vs ts `ref:from`
  - `delete_all`: ruby `ref:uniq` vs ts `ref:from`
- `activerecord/relation/query-methods.ts` — 3
  - `build_cast_value`: ruby `ref:name,ref:value,ref:defaultValue` vs ts `ref:name,ref:value,ref:constructor`
  - `arel_column`: ruby `ref:field` vs ts `ref:fieldStr`
- `activerecord/relation/predicate-builder.ts` — 2
  - `grouping_queries`: ruby `ref:queries` vs ts `ref:constructor`
  - `grouping_queries`: ruby `ref:queries` vs ts `ref:reduced`
- `activerecord/relation/calculations.ts` — 1
  - `execute_simple_calculation`: ruby `ref:relation,ref:columnName` vs ts `ref:relation,ref:aggregateTarget`
- `activerecord/relation/delegation.ts` — 1
  - `generate_relation_method`: ruby `ref:generatedRelationMethods,ref:method` vs ts `ref:name,ref:fn`
- `activerecord/relation/merger.ts` — 1
  - `merge`: ruby `ref:relation,ref:other` vs ts `ref:relation,ref:buildOther`
- `activerecord/relation/spawn-methods.ts` — 1
  - `except`: ruby `ref:except` vs ts `ref:exceptValues`
- `activerecord/scoping/default.ts` — 1
  - `build_default_scope`: ruby `ref:scopeObj` vs ts `ref:combinedScope`
- `activerecord/statement-cache.ts` — 1
  - `create`: ruby `ref:binds` vs ts `ref:normalizedBinds`

## Acceptance criteria

1. Every convergeable (`burndown` / `module-mixin-receiver`) `naming` row in the
   file set above is either converged to the Rails identifier, rewired to the
   `this`-typed mixin idiom, or re-filed as an a1/a3 finding against the RFC
   that owns the file — with the re-filed story id named in the PR body.
2. `pnpm parity:api:calls:args:report` shows this slot's convergeable count at
   **zero**; no row in the file set is added to any
   `call-mismatches-exclude/` shard (CLAUDE.md — converge, never ratify).
3. No public surface, method name, field name or behavior changes; the diff is
   locals and parameters only (plus mixin-receiver rewiring where it applies).
4. `pnpm build && pnpm test` green; `pnpm parity:api:calls:args` stays green.

## Notes for the claimer

The per-file counts above are from the 2026-08-21 parity artifact and are
**advisory**. Re-run
`API_COMPARE_FORCE=1 pnpm parity:api --calls && pnpm parity:api:calls:args:report`
at claim time and work from the fresh reading — counts drift as sibling RFCs
touch the same files.
