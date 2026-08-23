---
title: "Wave 5: burn down the AR-closure naming rows in activerecord associations"
status: done
updated: 2026-08-23
rfc: "0096-naming-identifier-burndown"
cluster: api-compare
packages: ["activerecord"]
deps: []
deps-rfc: []
est-loc: 80
priority: 13
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

15 rows across 6 files. **File set:** `activerecord/associations/**` only.

- `activerecord/associations/belongs-to-association.ts` — 4
  - `target_changed?`: ruby `ref:foreignKey` vs ts `ref:fk`
  - `target_previously_changed?`: ruby `ref:foreignKey` vs ts `ref:fk`
- `activerecord/associations/belongs-to-polymorphic-association.ts` — 3
  - `target_changed?`: ruby `ref:foreignType` vs ts `ref:foreignTypeName`
  - `target_previously_changed?`: ruby `ref:foreignType` vs ts `ref:foreignTypeName`
- `activerecord/associations/join-dependency.ts` — 3
  - `initialize`: ruby `ref:base,ref:table,ref:build` vs ts `ref:baseModel,ref:baseTable,ref:build`
  - `initialize`: ruby `ref:tree,ref:base` vs ts `ref:tree,ref:baseModel`
- `activerecord/associations/association.ts` — 2
  - `scope`: ruby `ref:scope` vs ts `ref:globalScope`
  - `target_scope`: ruby `ref:scopeForAssociation` vs ts `ref:sfa`
- `activerecord/associations/collection-association.ts` — 2
  - `target=`: ruby `ref:record,bool:true,kwargs{inversing=bool:true,replace=bool:true}` vs ts `ref:records,bool:true,kwargs{inversing=bool:true,replace=bool:true}`
  - `merge_target_lists`: ruby `ref:record` vs ts `ref:identity`
- `activerecord/associations/has-many-association.ts` — 1
  - `count_records`: ruby `ref:counterCacheColumn` vs ts `ref:column`

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
