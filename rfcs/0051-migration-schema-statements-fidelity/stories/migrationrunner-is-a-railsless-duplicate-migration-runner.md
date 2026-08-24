---
title: "MigrationRunner is a Rails-less duplicate of Migrator + SchemaMigration"
status: ready
updated: 2026-08-24
rfc: "0051-migration-schema-statements-fidelity"
cluster: null
deps: []
deps-rfc: []
est-loc: 150
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`packages/activerecord/src/migrator.ts` defines `MigrationRunner`, a second
migration runner with no Rails counterpart. It keeps its own
`MigrationEntry[]`, composes its own `schema_migrations` table name, builds its
own Arel table, and issues its own `CREATE TABLE IF NOT EXISTS` /
insert / delete SQL — duplicating what `Migrator` +
`SchemaMigration` (`packages/activerecord/src/migration.ts`,
`schema-migration.ts`) already do, and what Rails does solely through
`Migrator` and `SchemaMigration` (`migration.rb:1440+`,
`schema_migration.rb`).

It also takes `Migration[]` directly rather than `MigrationProxy[]`, so it has
no version/name identity of its own. PR #5525 made `Migration#version` return
`undefined` for an unversioned migration (Rails' `@version = nil`,
`migration.rb:799`), which forced a class-name fallback at the `MigrationRunner`
call site to keep its `schema_migrations` key non-null:

```ts
version: m.version ?? m.constructor.name,
```

That fallback is a trails invention living in a trails-invented class — it
should disappear with the class rather than be converged.

## Acceptance criteria

- [ ] Establish who still calls `MigrationRunner`; if it is dead or only
      reachable from trails-only paths, delete it and route callers through
      `Migrator` / `MigrationContext`.
- [ ] If it must stay, document why at the call site and drop the duplicated
      `schema_migrations` DDL/DML in favour of `SchemaMigration`.
- [ ] The `m.version ?? m.constructor.name` fallback is gone — a version-less
      migration is either rejected or carries a proxy-supplied version, as
      `MigrationProxy#load_migration` guarantees in Rails (`migration.rb:1195`).
