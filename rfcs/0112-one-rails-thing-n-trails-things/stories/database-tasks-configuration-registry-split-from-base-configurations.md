---
title: "DatabaseTasks.databaseConfiguration is a separate registry from Base.configurations"
status: in-progress
updated: 2026-08-25
rfc: "0112-one-rails-thing-n-trails-things"
cluster: split-stores
deps: []
deps-rfc: []
est-loc: 200
pr: 7050
claim: "2026-08-25T16:42:34Z"
assignee: "converge-clear-cache-lock-mysql-sqlite"
blocked-by: null
closed-reason: null
---

## Context

`DatabaseTasks.databaseConfiguration`
(`packages/activerecord/src/tasks/database-tasks.ts:55`) is a registry
independent of `Base.configurations`. Rails has no such field — every
`DatabaseTasks` lookup goes through `Base.configurations`
(`vendor/rails/activerecord/lib/active_record/tasks/database_tasks.rb:552`,
`def configs_for(**options) = Base.configurations.configs_for(**options)`).

The split forces call sites to diverge. `createCurrent`
(`database-tasks.ts:217-222`) must do `findDbConfig(envName)` and establish with
the resolved config object, because Rails' literal shape —
`migration_class.establish_connection(environment.to_sym)`
(`database_tasks.rb:653`) — would resolve the env string against
`Base.configurations` and can pick a different config.

`support/connection.ts:390-392` sets both registries together, but callers
reassign `databaseConfiguration` alone (e.g.
`packages/trailties/src/commands/db.test.ts:1162`, which swaps in a
production-env registry and restores only that field plus
`DatabaseConfigurations.current`), so the two genuinely diverge in practice.

Surfaced in review of PR #5507, which documented the divergence at the
`createCurrent` call site rather than converging it.

## Acceptance criteria

- [ ] Determine whether `DatabaseTasks.databaseConfiguration` can be retired in
      favour of `Base.configurations`, mirroring `database_tasks.rb:552`; if a
      distinct registry must stay, record why at its declaration.
- [ ] If retired, `createCurrent` establishes with the env string
      (`database_tasks.rb:653`) instead of a pre-resolved config object, and the
      call-site comment added by #5507 goes away.
- [ ] Callers that reassign `databaseConfiguration` alone (trailties `db.test.ts`
      and the AR task tests) are updated to the surviving registry.
- [ ] No test relies on the two registries holding different configs.

## Absorbed: `database-tasks-for-each-takes-built-configurations`

Merged in during the RFC 0023 triage pass (2026-08-18). Original title: "DatabaseTasks.forEach takes a built DatabaseConfigurations instead of constructing one"

### Context

Surfaced converging the `tasks/*` call-set rows in #6664; the
`for_each | new` row is baselined with a per-site reason rather than converged.

Rails, `vendor/rails/activerecord/lib/active_record/tasks/database_tasks.rb:141-154`:

```ruby
def for_each(databases) # :nodoc:
  return {} unless defined?(Rails)

  database_configs = ActiveRecord::DatabaseConfigurations.new(databases).configs_for(env_name: Rails.env)

  # if this is a single database application we don't want tasks for each primary database
  return if database_configs.count == 1

  database_configs.each do |db_config|
    next unless db_config.database_tasks?

    yield db_config.name
  end
end
```

`databases` is the raw railtie configuration hash; Rails wraps it in a fresh
`DatabaseConfigurations` on the spot. trails'
`DatabaseTasks.forEach(databases: DatabaseConfigurations, fn)`
(`packages/activerecord/src/tasks/database-tasks.ts`) requires the caller to
have built it, so the `new` never happens here. It also drops the
`db_config.database_tasks?` guard, yielding every config's name.

### Converged shape

Take the raw configuration hash and construct `DatabaseConfigurations` inside
the body, then filter on `databaseTasks` before yielding, matching `:150-153`.
`defined?(Rails)` has no trails analogue — leave that arm out and say so at the
call site, or fold it into the existing SKIP entry if one covers it.

### Acceptance criteria

- [ ] `forEach` constructs `DatabaseConfigurations` from its argument and honours
      the `database_tasks?` guard.
- [ ] Callers updated to pass the raw hash.
- [ ] Delete the `for_each | new` row from the exclude shard by hand via
      `serializeBaseline`, then
      `pnpm parity:api:calls:tighten activerecord/tasks/database-tasks.json`.
- [ ] `pnpm parity:api:calls` and `pnpm parity:api:calls:args` green.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
