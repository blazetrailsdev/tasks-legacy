---
title: "Inline SchemaDumper#table's column half, retiring emitTable"
status: ready
updated: 2026-08-24
rfc: "0051-migration-schema-statements-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 600
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

# Inline SchemaDumper#table's column half, retiring `emitTable`

## Context

Rails' `SchemaDumper#table` (`vendor/rails/activerecord/lib/active_record/schema_dumper.rb:157-229`)
emits the whole `create_table` block inline: it prints the `create_table` header,
resolves the primary key through `column_spec_for_primary_key` / `format_colspec`,
appends `format_options(table_options)`, then iterates `columns` calling
`valid_type?` and `column_spec` per column, and finally calls
`indexes_in_create`, `check_constraints_in_create`,
`exclusion_constraints_in_create` and `unique_constraints_in_create`.

The port splits that body in two. `SchemaDumper#table`
(`packages/activerecord/src/schema-dumper.ts:729-760`) fetches columns, indexes
and table options and then hands the whole emission half to an **abstract**
`emitTable`, implemented in
`packages/activerecord/src/connection-adapters/abstract/schema-dumper.ts:238-351`
(with per-dialect overrides under `connection-adapters/{mysql,postgresql,sqlite3}/schema-dumper.ts`).
`emitTable` / `emitTableBody` have no Rails counterpart at all. The port also
carries invented scaffolding around the same seam: `gatherInlineConstraints`,
`filterIndexesForDump`, `primaryKeyOrderCache` / `resolvePrimaryKeyColumns`.

Because the calls are made in a different FILE, the call-set gate cannot credit
them, and nine rows sit in
`scripts/api-compare/call-mismatches-exclude/activerecord/schema-dumper.json`
under `rubyName: "table"` (`column_spec`, `column_spec_for_primary_key`,
`format_colspec`, `format_options`, `table_options`, `valid_type?`,
`indexes_in_create`, `unique_constraints_in_create`,
`exclusion_constraints_in_create`), each pointing here.

`schema-dumper-emit-table-and-underscored-callee-convergence` (RFC 0106) converged
the other three real-divergence reasons in that cluster (`index_parts` →
`defaultIndexType`, `Migration#_run` → `run`, `_crc32` → `crc32`) and hands this
one over: inlining `emitTable` back into `table` is an architectural change
across five dumper files, well past that story's LOC ceiling.

## Acceptance criteria

- `SchemaDumper#table` emits the `create_table` block inline, in Rails' branch
  order, calling `columnSpec`, `columnSpecForPrimaryKey`, `formatColspec`,
  `formatOptions`, `tableOptions`, `validType`, `indexesInCreate`,
  `checkConstraintsInCreate`, `exclusionConstraintsInCreate` and
  `uniqueConstraintsInCreate` from its own body.
- `emitTable` / `emitTableBody` are deleted, along with the per-dialect
  overrides that exist only to serve them; dialect differences ride the Rails
  seams (`column_spec`, `table_options`, `prepare_column_options`) instead.
- The nine `rubyName: "table"` rows leave
  `scripts/api-compare/call-mismatches-exclude/activerecord/schema-dumper.json`
  by deletion, with `pnpm parity:api:calls:tighten activerecord/schema-dumper.json`
  for the shard.
- SQLite, PostgreSQL and MySQL dump output is unchanged (the existing
  `schema-dumper.test.ts` / per-dialect `schema-dumper.test.ts` suites stay green).
- Both call gates green.
