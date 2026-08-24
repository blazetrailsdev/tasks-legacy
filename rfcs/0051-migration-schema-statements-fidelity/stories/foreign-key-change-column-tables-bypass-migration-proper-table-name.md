---
title: "Change-column tables are created via direct createTable with decorated literals, bypassing Migration#properTableName"
status: ready
updated: 2026-08-24
rfc: "0051-migration-schema-statements-fidelity"
cluster: null
deps: []
deps-rfc: []
est-loc: 80
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Rails' `ForeignKeyChangeColumnTest` creates its tables with a real migration,
`CreateRocketsMigration`
(vendor/rails/activerecord/test/cases/migration/foreign_key_test.rb:34-43),
run via `@migration.migrate(:up)` / `(:down)` in setup/teardown. The table name
is decorated by `ActiveRecord::Migration#method_missing`, which routes the
first argument through `proper_table_name(arguments.first, table_name_options)`
— that is precisely what the `WithPrefix` / `WithSuffix` subclasses (:145-163)
are testing.

The port landed in #5543
(`packages/activerecord/src/migration/foreign-key.test.ts`,
`foreignKeyChangeColumnTest`) calls `conn.createTable` / `conn.dropTable`
directly with pre-decorated literals (`` `${prefix}rockets${suffix}` ``),
bypassing the migration layer. The prefix/suffix therefore never travels the
`Migration#method_missing` → `properTableName` path the subclasses exist to
cover; only the `TableDefinition#newForeignKeyDefinition` half of the
machinery is exercised.

`Migration.properTableName` and `Migration.tableNameOptions` already exist in
trails (`packages/activerecord/src/migration.ts:452`, `:1421`), so the port can
use a `SilentMigration` subclass the way the sibling
`CreateCitiesAndHousesMigration` / `CreateSchoolsAndClassesMigration` in the
same file already do, and pass the bare `rockets` / `astronauts` names.

## Acceptance criteria

- [ ] The change-column tables are created and dropped by a
      `CreateRocketsMigration` port (a `SilentMigration` subclass), not by
      direct `conn.createTable` / `conn.dropTable` calls.
- [ ] The migration is passed the _undecorated_ table names, with the
      prefix/suffix applied by `properTableName` / `tableNameOptions`, so the
      `WithPrefix` / `WithSuffix` variants cover that path.
- [ ] Green on all three adapters; `parity:test` delta non-negative and
      `--gates --check` stays at exit 0.
