---
title: "sqlite3-table-change-column-should-redeclare-the-column"
status: ready
updated: 2026-08-24
rfc: "0119-connection-adapter-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 60
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Rails `vendor/rails/activerecord/lib/active_record/connection_adapters/sqlite3/schema_definitions.rb`
`change_column` re-enters `column(column_name, type, **options)` on the **table
definition**. trails
`packages/activerecord/src/connection-adapters/sqlite3/schema-definitions.ts`
`changeColumn` forwards to the adapter's `changeColumn`, which rebuilds the
table via `copy_table`; the definition-level re-declaration never happens on the
definition object itself.

The two are not the same operation: Rails' `Table#change_column` mutates the
in-flight definition inside a `change_table` block, while a `copy_table` rebuild
is the adapter-level path. Surfaced by the RFC 0106 call-set gate (the `column`
row, now a CONVERGEABLE `@missingRailsCall` tag at the call site).

## Acceptance criteria

- [ ] SQLite `Table#changeColumn` calls `column(columnName, type, ...options)` on the definition, mirroring the Ruby body.
- [ ] The `@missingRailsCall column` tag on `sqlite3/schema-definitions.ts` is deleted, not re-justified.
- [ ] Rails' `change_table` / `change_column` test cases pass on the SQLite lane, and the PG/MySQL lanes stay green.
