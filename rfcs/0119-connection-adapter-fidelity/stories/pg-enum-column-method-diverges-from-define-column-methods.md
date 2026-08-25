---
title: "Abstract TableDefinition#enum is PG-only surface in Rails"
status: ready
updated: 2026-08-25
rfc: "0119-connection-adapter-fidelity"
cluster: null
deps: []
deps-rfc: []
est-loc: 120
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Re-scoped 2026-08-24 against `main` (152b2ebe9). This story originally named
three divergences; two have since converged and one is owned elsewhere:

- **Converged** — `enum` is now implemented on `PostgreSQL::TableDefinition`
  (`packages/activerecord/src/connection-adapters/postgresql/schema-definitions.ts:841-850`),
  where Rails puts it, rather than only declared in that file's `ColumnMethods`
  interface.
- **Converged** — it now passes `:enum` as the column type with the enum name
  riding along in options (`this.column(name, "enum", { ...rest, enumType })`),
  matching the generated body from
  `vendor/rails/activerecord/lib/active_record/connection_adapters/postgresql/schema_definitions.rb:185-189`.
- **Owned elsewhere** — the snake_case `enum_type` option key against the
  camelCase convention is the whole subject of
  `unify-enum-type-option-spelling-across-dsl-and-typetosql`, which covers the
  DSL, `typeToSql` and the dumper together. Not duplicated here.

What remains, and is not covered by any other story: **the abstract
`TableDefinition` still carries `enum`**
(`packages/activerecord/src/connection-adapters/abstract/schema-definitions.ts:1487`).

Rails declares `enum` only through PostgreSQL's `define_column_methods` call
(`postgresql/schema_definitions.rb:185-189`). `ActiveRecord::ConnectionAdapters::TableDefinition`
has no `enum` — a `t.enum` on a MySQL or SQLite table definition is a
`NoMethodError` in Rails and silently type-checks in trails.

This is the same shape as `drop-jsonb-from-abstract-table-definition`: PG-only
column surface that leaked onto the abstract definition.

## Acceptance criteria

- `enum` is removed from the abstract `TableDefinition` and its `ColumnMethods`
  interface; it survives only on the PostgreSQL subclass.
- A `t.enum` call on a non-PostgreSQL table definition no longer type-checks.
- Existing PG enum schema tests stay green on the PostgreSQL lane.
- `pnpm parity:api:extra --package activerecord` does not grow (the gate added
  by #6997 covers this file).
