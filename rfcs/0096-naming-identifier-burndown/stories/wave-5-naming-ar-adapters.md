---
title: "Wave 5: burn down the AR-closure naming rows in the activerecord connection adapters"
status: done
updated: 2026-08-23
rfc: "0096-naming-identifier-burndown"
cluster: api-compare
packages: ["activerecord"]
deps: []
deps-rfc: []
est-loc: 90
priority: 12
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

19 rows across 12 files. **File set:** `activerecord/connection-adapters/**` only.

- `activerecord/connection-adapters/postgresql-adapter.ts` — 4 (mmr 2)
  - `rename_table`: ruby `ref:newName` vs ts `ref:renamedName`
  - `create_schema_dumper`: ruby `ref:this,ref:options` vs ts `ref:source,ref:options`
- `activerecord/connection-adapters/abstract-mysql-adapter.ts` — 2
  - `build_insert_sql`: ruby `ref:first` vs ts `ref:firstKey`
  - `mismatched_foreign_key_details`: ruby `ref:sql` vs ts `ref:constructor`
- `activerecord/connection-adapters/abstract/connection-pool.ts` — 2
  - `checkout`: ruby `ref:acquireConnection` vs ts `ref:conn`
  - `checkout`: ruby `ref:acquireConnection` vs ts `ref:c`
- `activerecord/connection-adapters/postgresql/schema-definitions.ts` — 2
  - `new_exclusion_constraint_definition`: ruby `ref:name,ref:expression,ref:options` vs ts `ref:name,ref:expression,ref:opts`
  - `new_unique_constraint_definition`: ruby `ref:name,ref:columnName,ref:options` vs ts `ref:name,ref:columnName,ref:opts`
- `activerecord/connection-adapters/sqlite3-adapter.ts` — 2
  - `table_info`: ruby `ref:tableName` vs ts `ref:schema`
  - `table_info`: ruby `ref:tableName` vs ts `ref:bare`
- `activerecord/connection-adapters/abstract/connection-handler.ts` — 1
  - `resolve_pool_config`: ruby `ref:connectionName,ref:dbConfig,ref:role,ref:shard` vs ts `ref:ownerName,ref:dbConfig,ref:role,ref:shard`
- `activerecord/connection-adapters/abstract/query-cache.ts` — 1
  - `compute_if_absent`: ruby `ref:key` vs ts `ref:firstKey`
- `activerecord/connection-adapters/abstract/schema-definitions.ts` — 1
  - `rename_index`: ruby `ref:name,ref:indexName,ref:newIndexName` vs ts `ref:name,ref:oldName,ref:newName`
- `activerecord/connection-adapters/postgresql/database-statements.ts` — 1
  - `cast_result`: ruby `ref:fields,ref:values,ref:freeze` vs ts `ref:columnNames,ref:rows,ref:columnTypes`
- `activerecord/connection-adapters/postgresql/oid/point.ts` — 1
  - `build_point`: ruby `ref:Float,ref:Float` vs ts `ref:fx,ref:fy`
- `activerecord/connection-adapters/sqlite3/quoting.ts` — 1
  - `quoted_time`: ruby `ref:value` vs ts `ref:dt`
- `activerecord/connection-adapters/sqlite3/schema-statements.ts` — 1 (mmr 1)
  - `create_schema_dumper`: ruby `ref:this,ref:options` vs ts `ref:source,ref:options`

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
