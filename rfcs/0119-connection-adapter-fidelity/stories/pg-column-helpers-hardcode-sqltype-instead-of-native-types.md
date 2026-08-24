---
title: "PG column helpers stamp a hardcoded sqlType instead of resolving through nativeDatabaseTypes"
status: draft
updated: 2026-07-29
rfc: "0119-connection-adapter-fidelity"
cluster: null
deps: []
deps-rfc: []
est-loc: 180
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while porting the variadic `*names` shape in PR #5575.

Rails defines PostgreSQL's 31 column helpers purely through
`define_column_methods` (`vendor/rails/activerecord/lib/active_record/connection_adapters/postgresql/schema_definitions.rb:185-189`).
The generated body is
`names.each { |name| column(name, :#{column_type}, **options) }` — nothing more.
The SQL type is resolved later, at DDL time, by `type_to_sql` reading
`native_database_types` (`postgresql_adapter.rb` NATIVE_DATABASE_TYPES).

trails instead hand-writes all 31 as explicit methods in
`packages/activerecord/src/connection-adapters/postgresql/schema-definitions.ts`,
each routing through a private `pgColumn` that stamps a hardcoded literal
`sqlType` on the ColumnDefinition — `"BIGSERIAL"`, `"SERIAL"`, `` `BIT(${limit})` ``,
`` `BIT VARYING(${limit})` `` — and for the remaining 27 passes `undefined` and
relies on the type name alone.

Two consequences:

- The `bit` / `bitVarying` limit formatting duplicates logic that belongs in
  `typeToSql` / `nativeDatabaseTypes`, so the two can drift.
- A pre-stamped `sqlType` short-circuits the normal type resolution path, which
  is the same class of divergence already tracked by
  `converge-pg-build-change-column-definition-drop-prepopulated-sqltype`.

PR #5575 fixed the two side-effect bugs in this path (`index:` was dropped and
`raiseOnDuplicateColumn` was skipped) by routing `pgColumn` through
`TableDefinition#column`, but deliberately left the hardcoded `sqlType` stamping
in place — removing it needs the PG `nativeDatabaseTypes` entries verified first
and is larger than that PR's scope.

## Acceptance criteria

- [ ] PG column helpers no longer stamp a literal `sqlType`; each is the Rails
      one-liner `column(name, <type>, options)` per name.
- [ ] `bigserial`, `serial`, `bit`, `bitVarying` resolve their SQL type through
      `nativeDatabaseTypes` / `typeToSql`, including the `limit` formatting for
      the two bit types.
- [ ] `packages/activerecord/src/connection-adapters/postgresql/schema-definitions.test.ts`'s
      "keeps type-specific SQL types when defining multiple columns" still passes
      (adjusted to assert emitted DDL rather than the stamped field if needed).
- [ ] Schema-dumper and migration PG suites green on all three lanes.
- [ ] parity:api delta non-negative; the 31 helpers ideally collapse to the
      generated `defineColumnMethods` path.
