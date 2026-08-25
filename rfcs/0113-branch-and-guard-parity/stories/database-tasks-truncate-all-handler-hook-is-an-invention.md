---
title: "DatabaseTaskHandler.truncateAll is a trails invention; mysql/sqlite tasks hand-roll what the adapter already emits"
status: ready
updated: 2026-08-25
rfc: "0113-branch-and-guard-parity"
cluster: missing-arm
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

Surfaced while shipping `pg-database-tasks-truncate-all-is-an-invention` (#6158),
which deleted `PostgreSQLDatabaseTasks#truncateAll` and gave
`DatabaseTasks.truncateTables` Rails' body. The per-adapter _handler hook_ it
routes around is still there and is a trails invention.

Rails has exactly one truncation path. `DatabaseTasks#truncate_tables`
(`activerecord/lib/active_record/tasks/database_tasks.rb:230-234`) is:

```ruby
def truncate_tables(db_config)
  with_temporary_connection(db_config) do |conn|
    conn.truncate_tables(*conn.tables)
  end
end
private :truncate_tables

def truncate_all(environment = env)
  configs_for(env_name: environment).each { |db_config| truncate_tables(db_config) }
end
```

The adapter, not the task class, owns the statement:
`truncate_tables` (`abstract/database_statements.rb:222-231`) subtracts the
schema_migrations / ar_internal_metadata names, wraps
`disable_referential_integrity`, and batches
`build_truncate_statements`. No `*DatabaseTasks` class in
`activerecord/lib/active_record/tasks/` defines `truncate_all` at all.

trails instead has an optional `truncateAll?(config)` member on
`DatabaseTaskHandler` (`packages/activerecord/src/tasks/database-tasks.ts:1451`)
that both `DatabaseTasks.truncateAll` (`:529-540`) and
`DatabaseTasks.truncateTables` (`:1407-1425`) check _first_, falling through to
the Rails path only when a handler defines none. Two handlers still ride on it,
each hand-rolling the table discovery and statement Rails asks the adapter for:

- `packages/activerecord/src/tasks/mysql-database-tasks.ts:151-...` — its own
  `truncateAll`, registered at `:187`
- `packages/activerecord/src/tasks/sqlite-database-tasks.ts:222-...` — its own
  `truncateAll` (DELETE FROM + sqlite_sequence reset), registered at `:319`

Both duplicate adapter behaviour that already exists: `SQLite3Adapter`
overrides `buildTruncateStatement` (`sqlite3-adapter.ts:1283`) and MySQL
inherits the abstract `truncateTables`, so the adapter path already emits the
right statements for both.

## Converged shape

`DatabaseTaskHandler.truncateAll` is deleted, with the `mysql-database-tasks.ts`
and `sqlite-database-tasks.ts` methods and their `registerTask` entries.
`DatabaseTasks.truncateTables` keeps only the Rails body (the
`withTemporaryConnection` arm #6158 added) and `DatabaseTasks.truncateAll`
becomes the plain `configs_for(...).each { truncate_tables(...) }` loop
(`database_tasks.rb:237-241`).

Note the two `DatabaseTasksTruncateAllTest` / `...WithMultipleDatabasesTest`
suites in `packages/activerecord/src/tasks/database-tasks.test.ts:1160-1245`
register a fake `abstract` handler _with_ a `truncateAll` and assert it is
called — they are ports of Rails tests that stub `truncate_tables` itself, so
they need re-pointing at the connection path (Rails'
`activerecord/test/cases/tasks/database_tasks_test.rb`, `DatabaseTasksTruncateAllTest`)
rather than deleting. `database-tasks-truncate-tables.trails.test.ts` (added by PR 6158) already covers the connection path and should keep passing untouched.

## Acceptance criteria

- [ ] `truncateAll?` is gone from `DatabaseTaskHandler`; no task class defines
      `truncateAll`; no `truncateAll:` key in any `registerTask` call.
- [ ] `DatabaseTasks.truncateTables` / `truncateAll` are the Rails bodies with
      no handler branch (`database_tasks.rb:230-241`).
- [ ] The `DatabaseTasksTruncateAllTest` suites keep their Rails names and
      exercise the connection path.
- [ ] sqlite, mysql and pg lanes green.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0113-branch-and-guard-parity`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
