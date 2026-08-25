---
title: "registerTask(/postgres/) is wider than Rails' /postgresql/"
status: draft
updated: 2026-08-13
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 40
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced in #6487, which flipped the task registry from handler singletons to
task classes and so put the `register_task` pattern list side by side with
Rails' for the first time.

Rails registers four patterns
(`vendor/rails/activerecord/lib/active_record/tasks/database_tasks.rb:78-81`):

```ruby
register_task(/mysql/,        "ActiveRecord::Tasks::MySQLDatabaseTasks")
register_task(/trilogy/,      "ActiveRecord::Tasks::MySQLDatabaseTasks")
register_task(/postgresql/,   "ActiveRecord::Tasks::PostgreSQLDatabaseTasks")
register_task(/sqlite/,       "ActiveRecord::Tasks::SQLiteDatabaseTasks")
```

trails registers `/postgres/`
(`packages/activerecord/src/tasks/postgresql-database-tasks.ts`, `register()`),
which is strictly wider than Rails': it also matches an adapter spelled
`postgres`, where Rails' `adapter[/postgresql/]` misses and `class_for_adapter`
raises `DatabaseNotSupported` (`:574-580`). `/mysql/` and `/sqlite/` match
Rails exactly.

The `/trilogy/` registration is deliberately absent — trilogy is Ruby-only and
out of scope for the port — so this story covers the postgresql pattern alone.

## Converged shape

`PostgreSQLDatabaseTasks.register()` registers `/postgresql/`. Any trails
caller or test relying on a bare `postgres` adapter name resolving to the task
class is either renamed to `postgresql` or, if it is exercising a real Rails
adapter alias, tracked separately with the alias' Rails citation.

## Acceptance criteria

1. The registered pattern is `/postgresql/`, matching `database_tasks.rb:80`.
2. An adapter named `postgres` raises `DatabaseNotSupported` with Rails'
   message, as `class_for_adapter` (`:574-580`) does for any unmatched adapter.
3. The `tasks/` and postgresql rake suites stay green.
