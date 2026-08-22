---
title: "SQLite addColumn reimplements the ALTER instead of calling super"
status: ready
updated: 2026-08-22
rfc: "0112-one-rails-thing-n-trails-things"
cluster: duplicate-bodies
deps: []
deps-rfc: []
est-loc: 150
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

# SQLite addColumn reimplements the ALTER instead of calling super

## Context

Surfaced by PR #5568 while removing the SQLite adapter's private `typeToSql`
shadow.

Rails' `SQLite3Adapter#add_column`
(`vendor/rails/activerecord/lib/active_record/connection_adapters/sqlite3_adapter.rb:338-347`)
is four lines: it routes an `invalid_alter_table_type?` column through
`alter_table`, and otherwise calls **`super`** — the abstract
`SchemaStatements#add_column`, which builds the column through
`schema_creation` / `add_column_for_alter` and so shares one code path with
`create_table`.

trails' `addColumn`
(`packages/activerecord/src/connection-adapters/sqlite3-adapter.ts`, ~:1664)
instead hand-assembles the `ALTER TABLE … ADD COLUMN` string: it re-derives the
SQL type via a private `_baseColumnType`, then concatenates `COLLATE`,
`GENERATED ALWAYS AS`, `NOT NULL` and `DEFAULT` clauses itself.

Consequence: the add-column path and the create-table path render columns
through different code, so they can disagree. PR #5568 already hit one instance
— the private `typeToSql` shadow uppercased its output and then rejected
`DATETIME(6)` as an invalid type, while `SchemaCreation#typeToSql` returned it
verbatim as Rails does. That shadow is gone, but the duplicated clause assembly
around it remains, and `_baseColumnType` has no Rails counterpart at all.

Note the interaction with
`0005-activerecord-gaps/converge-type-to-sql-base-names-on-native-database-types`:
that story changes what `typeToSql` returns, and this path is one of its
consumers. Sequence them, or expect to touch the same lines twice.

## Acceptance criteria

- [ ] SQLite `addColumn` keeps only the `isInvalidAlterTableType` branch and
      otherwise delegates to the abstract `SchemaStatements#addColumn`, matching
      `sqlite3_adapter.rb:338-347`.
- [ ] `_baseColumnType` and the hand-rolled clause concatenation are deleted, or
      what survives is justified at the call site against Rails.
- [ ] The `t.virtual` declared-type behaviour `_baseColumnType` encodes is
      preserved (it is exercised by the generated-column tests).
- [ ] parity:api delta non-negative; green on all three lanes.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
