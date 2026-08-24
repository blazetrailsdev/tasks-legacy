---
title: "trailties db migrate:status re-implements DatabaseTasks.migrateStatus instead of calling it"
status: ready
updated: 2026-08-24
rfc: "0051-migration-schema-statements-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 90
priority: 56
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Rails' `db:migrate:status` task is three lines — it delegates the whole body to
`DatabaseTasks.migrate_status`
(`vendor/rails/activerecord/lib/active_record/railties/databases.rake:229-233`):

```ruby
task status: :load_config do
  ActiveRecord::Tasks::DatabaseTasks.with_temporary_pool_for_each do
    ActiveRecord::Tasks::DatabaseTasks.migrate_status
  end
end
```

trails has that method — `DatabaseTasks.migrateStatus`
(`packages/activerecord/src/tasks/database-tasks.ts:1020-1040`), a faithful port
of `tasks/database_tasks.rb:302-315` including the missing-table guard, the
`database:` header, the `center`/`ljust` column widths and the separator rule.

But `packages/trailties/src/commands/db.ts`'s `migrate:status` action does not
call it. It re-derives the same output inline against its own
`MigrationContext`: its own `schemaMigration.tableExists()` abort, its own
header, its own `padEnd` widths, its own `"  up  "` / `" down "` literals. The
two implementations already disagree on column widths and on the `database:`
header line, which trailties omits entirely.

The same applies to `db version`, whose Rails counterpart
(`databases.rake:308-313`) prints a `database:` line trailties also omits.

This is the shape RFC 0051's
`database-tasks-rake-only-migration-methods-move-to-callers` was about, run in
reverse: `migrate_status` is one of the methods Rails _keeps_ on `DatabaseTasks`
and calls from the rake task, so the caller should call it rather than inline it.

Surfaced in PR #6982, which repointed this action from `Migrator#migrationsStatus`
onto `MigrationContext#migrationsStatus` and added the missing-table abort —
converging the _data_ source while leaving the duplicated presentation in place.

Coordinate with the in-progress story
`migrate-status-aborts-as-error-and-invents-a-memory-database-fallback`, which
is fixing `DatabaseTasks.migrateStatus`'s own two divergences (a catchable
`Error` where Rails uses `Kernel.abort`, and an invented `":memory:"` database
fallback). Landing that first means this story inherits the corrected body
instead of duplicating a wrong one — prefer taking it in that order.

## Converged shape

`db migrate:status`'s action becomes the trails spelling of
`databases.rake:229-233`: iterate the configs and call
`DatabaseTasks.migrateStatus()`, with no local header, widths, status literals
or table-exists guard. The multi-database `[name]` prefix stays wherever
trailties' `withPrefixedStdout` already provides it.

## Acceptance criteria

- [ ] `packages/trailties/src/commands/db.ts`'s `migrate:status` action calls
      `DatabaseTasks.migrateStatus()` and holds no duplicate presentation logic.
- [ ] The missing-table arm comes from `DatabaseTasks.migrateStatus`
      (`database_tasks.rb:303-305`), not from a second guard in `db.ts`.
- [ ] Output matches `database_tasks.rb:308-314`, including the `database:`
      header line trailties currently omits.
- [ ] `packages/trailties/src/commands/db.test.ts` keeps its test names; any
      assertion that encoded the old widths is updated to the Rails ones.
