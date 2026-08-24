---
title: "DatabaseTasks carries five rake-only migration methods Rails has no counterpart for"
status: done
updated: 2026-08-24
rfc: "0051-migration-schema-statements-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 300
priority: null
pr: 6977
claim: "2026-08-24T11:45:44Z"
assignee: "boot-dump-fingerprint-misses-mysql-functional-index"
blocked-by: null
closed-reason: null
---

## Context

`DatabaseTasks` carries five methods Rails does not have, each already marked
`@internal` with the rake line it stands in for
(`packages/activerecord/src/tasks/database-tasks.ts`):

- `rollback(steps)` / `forward(steps)` and their `_stepMigrations` helper —
  `rake db:rollback` / `db:forward` inline
  `DatabaseTasks.migration_connection_pool.migration_context.rollback(step)`
  (`vendor/rails/activerecord/lib/active_record/railties/databases.rake:269`,
  `:279`).
- `runMigration(direction, version)` — `db:migrate:up` / `db:migrate:down`
  inline `migration_connection_pool.migration_context.run(direction,
target_version)` (`databases.rake:174-177`, `:205-208`).
- `migrateStatus()` — `db:migrate:status` inlines
  `migration_connection_pool.migration_context.migrations_status.each { … }`
  plus the table printing (`databases.rake` `:status`; the AR-side reader is
  `tasks/database_tasks.rb:311`).
- `currentVersion()` — `db:version` inlines
  `migration_connection_pool.migration_context.current_version`.

The justification recorded at each call site was that the body needed a pool
handle only `DatabaseTasks` had. PR #6974 removed that constraint: every one of
these is now a one-line call on `pool.migrationContext`, and the pool is
reachable from the CLI through `DatabaseTasks.migrationConnectionPool()`.

## Converged shape

The five methods are deleted from `DatabaseTasks`. Their bodies move into the
rake-task ports that call them — trailties `packages/trailties/src/commands/db.ts`
and `packages/activerecord-cli/src/db-tasks.ts` — each becoming the same one or
two lines the rake task is, over `DatabaseTasks.migrationConnectionPool()
.migrationContext`. The `migrate_status` table formatting travels with
`db:migrate:status`, where `databases.rake` puts it.

## Acceptance criteria

- [ ] `DatabaseTasks.rollback`, `forward`, `_stepMigrations`, `runMigration`,
      `migrateStatus`, `currentVersion` are gone.
- [ ] Each caller inlines the `migration_context` call its rake task inlines,
      cited to the `databases.rake` line.
- [ ] `pnpm parity:api:extra --package activerecord` shows a lower novel count
      for `tasks/database-tasks.ts`.
- [ ] trailties `db` command tests and the activerecord-cli db tests keep their
      names and pass.
