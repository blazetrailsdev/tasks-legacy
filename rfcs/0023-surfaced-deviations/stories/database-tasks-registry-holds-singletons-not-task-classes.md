---
title: "@tasks holds handler singletons, so database_adapter_for cannot instantiate or forward *arguments"
status: draft
updated: 2026-08-07
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 290
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced in #6185. `class_for_adapter` and `database_adapter_for` were ported
onto their Rails names there, but the registry they read is a trails invention
and the Rails bodies could only be matched down to their lookup half.

Rails, `vendor/rails/activerecord/lib/active_record/tasks/database_tasks.rb`:

```ruby
# :73-76
def register_task(pattern, task)
  @tasks ||= {}
  @tasks[pattern] = task
end

register_task(/mysql/,        "ActiveRecord::Tasks::MySQLDatabaseTasks")   # :78
register_task(/trilogy/,      "ActiveRecord::Tasks::MySQLDatabaseTasks")   # :79
register_task(/postgresql/,   "ActiveRecord::Tasks::PostgreSQLDatabaseTasks") # :80
register_task(/sqlite/,       "ActiveRecord::Tasks::SQLiteDatabaseTasks")  # :81

# :566-572
def database_adapter_for(db_config, *arguments)
  klass = class_for_adapter(db_config.adapter)
  converted = klass.respond_to?(:using_database_configurations?) && klass.using_database_configurations?
  config = converted ? db_config : db_config.configuration_hash
  klass.new(config, *arguments)
end

# :574-580
def class_for_adapter(adapter)
  _key, task = @tasks.reverse_each.detect { |pattern, _task| adapter[pattern] }
  unless task
    raise DatabaseNotSupported, "Rake tasks not supported by '#{adapter}' adapter"
  end
  task.is_a?(String) ? task.constantize : task
end
```

Three divergences remain in `packages/activerecord/src/tasks/database-tasks.ts`:

1. **`@tasks` is a Hash of pattern => task CLASS (or its name); ours is an
   array of pattern => handler SINGLETON.** So `database_adapter_for` cannot
   do `klass.new(config, *arguments)` — the handler already is the instance —
   and `using_database_configurations?` / `configuration_hash` (the arm that
   decides whether the task gets a `DatabaseConfig` or a raw hash) has no
   analogue at all. Every trails task handler unconditionally receives a
   `DatabaseConfig`.
2. **`*arguments` is dropped throughout.** `create`, `drop`, `charset`,
   `collation`, `structure_dump`, `structure_load` all take `(configuration,
*arguments)` and forward them to the task constructor. Ours take only the
   configuration (and `structureDump`/`structureLoad` take an invented
   `extraFlags` parameter instead of Rails' `filename = arguments.delete_at(0)`
   positional destructuring).
3. **`resolveTask` is invented public surface** duplicating
   `class_for_adapter`'s `reverse_each.detect` half. `pnpm parity:api:extra --package
activerecord` lists it among the file's novel names. `classForAdapter`
   currently delegates to it; deleting it means rewriting six call sites in
   `database-tasks.test.ts` (:220, :228, :232), `support/connection.test.ts`
   (:112), `mysql-database-tasks.test.ts` (:44),
   `postgresql-database-tasks.test.ts` (:46), `sqlite-database-tasks.test.ts`
   (:86) — the "unregistered task" one expects `undefined` where
   `classForAdapter` raises.

## Converged shape

`registerTask(pattern, task)` stores the task CLASS keyed by pattern;
`classForAdapter` folds the `reverse_each.detect` inline and `resolveTask` is
deleted; `databaseAdapterFor(dbConfig, ...arguments_)` reinstates the
`usingDatabaseConfigurations` / `configurationHash` arm and constructs
`new klass(config, ...arguments_)`. The `*arguments` thread is then restorable
through `create`/`drop`/`charset`/`collation`/`structureDump`/`structureLoad`,
retiring the invented `extraFlags` parameter.

Likely needs splitting — the registry flip touches all four task handler
modules (`sqlite-`, `postgresql-`, `mysql-database-tasks.ts` each expose a
`register()` that pushes a singleton) plus their tests. Size accordingly when
triaged; estimate below is the registry flip alone.

## Acceptance criteria

- `@tasks` holds task classes; `databaseAdapterFor` instantiates and carries
  the `using_database_configurations?` arm.
- `resolveTask` is gone and `parity:api:extra --package activerecord` shows one fewer
  novel name on `tasks/database-tasks.ts`.
- `*arguments` reaches the task constructor from every Rails entry point that
  declares it; `extraFlags` is deleted.
- Handler test names stay verbatim; the `unregistered task` case asserts
  `DatabaseNotSupported` with Rails' message.
- `pnpm parity:api:calls` stays green.

## Update 2026-08-09 (PR #6270)

Two new receipts landed on this deviation. `DatabaseTasks.charset` and
`#collation` are bare sends in Ruby
(`activerecord/lib/active_record/tasks/database_tasks.rb:332-335, 342-345`) —
a task class defining neither answers `NoMethodError`, which is exactly what
`SqliteDBCollationTest#test_db_retrieves_collation`
(`test/cases/adapters/sqlite3/sqlite_rake_test.rb:159-163`) asserts, since
`SQLiteDatabaseTasks` has no `collation`.

Because the registry holds a plain object whose members are all optional,
`packages/activerecord/src/tasks/database-tasks.ts` now has to raise that
`NoMethodError` by hand at both sites:

```ts
if (!handler.collation) {
  throw new NoMethodError(
    `undefined method 'collation' for an instance of ${handler.constructor.name}`,
  );
}
return handler.collation(config);
```

The message is also wrong in a way only this deviation causes: a handler
registered as an object literal has `constructor.name === "Object"`, so it
reads `for an instance of Object` where Rails reads
`for an instance of ActiveRecord::Tasks::SQLiteDatabaseTasks`.

Flipping the registry to task **classes** retires both guards and both
messages: `databaseAdapterFor(...).collation` becomes a real missing method on
a real class, so the language raises with the class's own name and the two
`if (!handler.x)` blocks and the `NoMethodError` import are deleted. Add that
to the acceptance criteria when this is triaged.

## Absorbed: `database-tasks-dispatch-guards-noop-where-rails-raises`

Merged in during the RFC 0023 triage pass (2026-08-18). Original title: "create/drop/purge no-op where Rails raises NoMethodError"

### Context

Surfaced in #6487, which converged `database_adapter_for` onto Rails'
`klass.new(config, *arguments)` (`database_tasks.rb:566-572`) and the dispatch
sites onto Rails' bare no-arg sends.

Rails sends unconditionally. `create` is `database_adapter_for(db_config,
*arguments).create` (`:117`); `drop` is the same at `:212`; `purge` at `:349`;
`truncate_tables` reaches the pool, not a task method (`:387-395`). A task class
defining none of these answers `NoMethodError`, which propagates.

trails guards every one of them
(`packages/activerecord/src/tasks/database-tasks.ts`, in `create`, `drop`,
`purge`, `truncateAll` and `truncateTables`):

```ts
const handler = this.databaseAdapterFor(dbConfig);
if (handler.create) {
  await handler.create();
}
```

so a task class missing the method is silently a no-op — and `create` then goes
on to print its `Created database '...'` banner (`:119`) for a database it never
created. `charset` and `collation` already take the other arm and raise
`NoMethodError` by hand with Rails' message, which is the shape the rest should
match; `SqliteDBCollationTest#test_db_retrieves_collation`
(`vendor/rails/activerecord/test/cases/adapters/sqlite3/sqlite_rake_test.rb:159-163`)
is the test that pins it.

The guards exist because `DatabaseTaskInstance`'s members are all optional —
TS has to model an absence Ruby gets for free — but "optional in the type" does
not have to mean "skipped at runtime".

### Converged shape

Each dispatch site raises `NoMethodError` with Rails' message when the
constructed task instance does not define the method, exactly as `charset` and
`collation` already do. `truncateAll` keeps its existing fallback only if that
fallback is itself Rails-backed; otherwise it is folded into the story that
tracks it (`database-tasks-truncate-all-handler-hook-is-an-invention`).

### Acceptance criteria

1. `create`, `drop` and `purge` raise `NoMethodError` for a task class that
   does not define the method, instead of no-opping and (for `create`) printing
   a success banner.
2. The message matches the one `charset` / `collation` already emit.
3. The `tasks/` suites stay green, and the banner tests still assert the
   banners for task classes that DO define the method.

## Triage note (2026-08-18)

Merged story (~290 LOC est.). One root cause, two symptoms: because `@tasks`
holds handler _singletons_ rather than task _classes_, `database_adapter_for`
cannot do Rails' `klass.new(config, *arguments)` (`database_tasks.rb:566-572`)
— and that is precisely why the `create` / `drop` / `purge` dispatch sites
(`:117`, `:212`, `:349`) guard instead of sending unconditionally and letting
`NoMethodError` propagate. Fixing the registry is what makes the bare sends
possible, so they converge in one PR.
