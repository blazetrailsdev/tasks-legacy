---
title: "trailties and activerecord-cli scan migrations a second time; the rake tasks do not"
status: ready
updated: 2026-08-24
rfc: "0051-migration-schema-statements-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 200
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

trailties discovers migrations a second time purely to decide whether to print
`"No migrations found."` before delegating, and activerecord-cli keeps a
`loadMigrations` helper for the same shape:

- `withMigrationTasksForDb` (`packages/trailties/src/commands/db.ts:592-598`),
  `runMigrateAll` (`:639-648`) and the `db:migrate:status` arm (`:1045-1050`)
  each call `discoverMigrations(...)` and return early on an empty list.
- `loadMigrations` (`packages/activerecord-cli/src/db-helpers.ts:52-60`) builds
  its own `MigrationContext` with `NullSchemaMigration` /
  `NullInternalMetadata` for `checkPendingMigrations`
  (`packages/activerecord-cli/src/pending-migrations.ts:46`).

Rails' rake tasks have neither. `db:migrate` is
`ActiveRecord::Tasks::DatabaseTasks.migrate_all` followed by `db:_dump`
(`vendor/rails/activerecord/lib/active_record/railties/databases.rake:88-92`) —
no pre-flight scan, no early return, and an empty `migrations_paths` simply
runs nothing. `db:abort_if_pending_migrations`
(`databases.rake:308-317`) reads
`ActiveRecord::Base.connection_pool.migration_context.needs_migration?` /
`pending_migration_versions` rather than assembling a list itself. After
PR #6974 these are the last places that discover migrations outside the pool's
`migration_context`, and each is one extra full directory scan per command.

## Converged shape

The pre-flight `discoverMigrations` calls and their `"No migrations found."`
early returns are deleted; the tasks delegate unconditionally as the rake tasks
do. `loadMigrations` is deleted and `checkPendingMigrations` reads
`DatabaseTasks.migrationConnectionPool().migrationContext
.pendingMigrationVersions()`, the reader `databases.rake:308-317` uses. If a
user-facing "nothing to run" line has to survive for CLI ergonomics, it is
driven off what the delegated call reports, not off a second scan.

## Acceptance criteria

- [ ] No `discoverMigrations` pre-flight guard remains in
      `packages/trailties/src/commands/db.ts`.
- [ ] `loadMigrations` is gone from `packages/activerecord-cli/src/db-helpers.ts`;
      `checkPendingMigrations` goes through the pool's `migration_context`.
- [ ] trailties `db` command tests and the activerecord-cli db tests keep their
      names and pass.
