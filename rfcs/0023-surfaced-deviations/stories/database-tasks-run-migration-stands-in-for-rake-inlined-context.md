---
title: "DatabaseTasks.runMigration/rollback/forward stand in for rake-inlined migration_context calls"
status: draft
updated: 2026-08-02
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
  - "trailties"
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

Rails has no `DatabaseTasks.run_migration`. `db:migrate:up` and
`db:migrate:down` inline
`ActiveRecord::Tasks::DatabaseTasks.migration_connection_pool.migration_context.run(:up, target_version)`
in the rake task itself
(`activerecord/lib/active_record/railties/databases.rake:174-177` and
`:205-208`).

PR #5864 added `DatabaseTasks.runMigration(direction, version)`
(`packages/activerecord/src/tasks/database-tasks.ts`, marked `@internal`)
because the trailties CLI has no pool handle of its own to reach
`migration_context` through. This is the same deviation already carried by
`DatabaseTasks.rollback` / `forward`, added by PR #5616 for
`databases.rake:269` / `:279` — three trails-only task methods now stand in
for rake-inlined bodies.

Converging means giving the CLI the pool/`MigrationContext` handle Rails' rake
tasks have, then inlining the three bodies at their call sites in
`packages/trailties/src/commands/db.ts` and deleting the `DatabaseTasks`
methods. Doing all three together keeps the seam consistent rather than
half-converged.

## Acceptance criteria

- [ ] The CLI reaches `migrationConnectionPool().migrationContext` (or the
      trails equivalent) directly, as `databases.rake` does.
- [ ] `DatabaseTasks.runMigration`, `rollback` and `forward` are deleted, or
      the ones that must stay are justified against `databases.rake`.
- [ ] `db migrate:up` / `migrate:down` / `rollback` / `forward` behavior is
      unchanged, including `migrate:up --version=<unknown>` still raising
      `UnknownMigrationVersionError` when no migration files exist.
