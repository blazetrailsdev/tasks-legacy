---
title: "DatabaseTasks.forEach is callerless while trailties hand-rolls its own multi-db fan-out"
status: ready
updated: 2026-08-25
rfc: "0112-one-rails-thing-n-trails-things"
cluster: null
packages: []
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

`DatabaseTasks.forEach` (`packages/activerecord/src/tasks/database-tasks.ts`)
has **zero callers** in trails. Confirmed by grep across `packages/` at the
merge of PR #7050, which converged its body onto Rails
(`tasks/database_tasks.rb:141-154`) — the body is now faithful, but nothing
invokes it.

In Rails it is the fan-out that generates the per-database namespaced tasks.
`activerecord/lib/active_record/railties/databases.rake` calls it five times —
`:35`, `:54`, `:105`, `:120` (plus `with_temporary_pool_for_each` at `:123`) —
each time as:

```ruby
ActiveRecord::Tasks::DatabaseTasks.for_each(databases) do |name|
  # define db:create:<name>, db:drop:<name>, db:migrate:<name>, ...
end
```

`databases` there is the raw railtie configuration hash, which is why
`for_each` wraps it in a fresh `DatabaseConfigurations` itself (`:144`) and
skips configs failing `db_config.database_tasks?` (`:150`).

trails' `packages/trailties/src/commands/db.ts` instead hand-rolls its own
fan-out (`forEachDatabase` / `forEachDatabaseConfig`, `:169` and `:196`), which
resolves configs on its own and does not go through `DatabaseTasks.forEach` or
honour `databaseTasks()`.

So there are two multi-database fan-outs: Rails' ported one, unused, and a
trails-invented one doing the real work — the "one Rails thing, N trails
things" shape this RFC exists to collapse.

## Converged shape

Route `db.ts`'s per-database command fan-out through `DatabaseTasks.forEach`,
passing the raw configuration hash the way `databases.rake` does, and retire
`forEachDatabase` / `forEachDatabaseConfig` (or reduce them to thin wrappers
that only add the adapter checkout `forEach` has no counterpart for).

Note `for_each`'s `return {} unless defined?(Rails)` guard (`:142`) has no
trails analogue and is deliberately absent — that is already documented at the
call site and is not part of this convergence.

## Acceptance criteria

- [ ] The trailties per-database command fan-out goes through
      `DatabaseTasks.forEach`.
- [ ] `databaseTasks()`-disabled and replica configs are skipped by the
      surviving path, matching `:150`.
- [ ] `forEachDatabase` / `forEachDatabaseConfig` are retired or reduced to the
      adapter-checkout concern only.
- [ ] `DatabaseTasks.forEach` has at least one production caller.
- [ ] trailties `db.test.ts` green.
