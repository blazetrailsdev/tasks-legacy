---
title: 'charset_current/collation_current hardcode "primary" and drop Rails'' db_name parameter'
status: draft
updated: 2026-08-07
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 90
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while converging the `tasks/database-tasks.ts` unrouted-privates
cluster in #6185 (`activerecord-unrouted-privates-tasks-and-migration`). The
cluster's own eight privates are converged; these two `*_current` readers were
adjacent and deliberately left alone to keep that PR scoped.

Rails, `vendor/rails/activerecord/lib/active_record/tasks/database_tasks.rb`:

```ruby
# :327-330
def charset_current(env_name = env, db_name = name)
  db_config = configs_for(env_name: env_name, name: db_name)
  charset(db_config)
end

# :337-340
def collation_current(env_name = env, db_name = name)
  db_config = configs_for(env_name: env_name, name: db_name)
  collation(db_config)
end
```

`configs_for(env_name:, name:)` returns the ONE config for that pair (the
singular form — `database_configurations.rb` returns a single DatabaseConfig
when `name:` is given, not an array).

Ours (`packages/activerecord/src/tasks/database-tasks.ts`, `charsetCurrent` /
`collationCurrent`) instead does:

```ts
const configs = this.configsFor(env);
if (configs.length === 0) return null;
const primary = configs.find((c) => c.name === "primary") ?? configs[0];
return this.charset(primary);
```

Three divergences: the `db_name` parameter is missing entirely (so a
multi-database app cannot ask for a non-primary database's charset); the
`"primary"` literal is hardcoded where Rails defaults to `DatabaseTasks.name`;
and the `configs[0]` fallback plus the `null`-on-empty arm are inventions —
Rails lets `configs_for` raise `AdapterNotSpecified` when the pair does not
resolve.

## Converged shape

```ts
static async charsetCurrent(envName?: string, dbName?: string): Promise<string | null> {
  const dbConfig = this.configsFor(...);  // env_name + name, singular
  return this.charset(dbConfig);
}
```

with `dbName` defaulting to `DatabaseTasks.name` (the same accessor Rails'
`name` reads) rather than the `"primary"` literal, and the same for
`collationCurrent`. `charset`/`collation` themselves already take Rails'
`configuration` and route through `resolveConfiguration`/`databaseAdapterFor`
after #6185, so only the two `*_current` readers move.

## Acceptance criteria

- `charsetCurrent`/`collationCurrent` take `(envName, dbName)` with Rails'
  defaults and resolve one config through the `env_name:`/`name:` pair.
- The `configs[0]` fallback and the empty-list `null` arm are deleted.
- `pnpm parity:api:calls` stays green; any newly-matched body gets a reviewed reason.
- Rails' `DatabaseTasksCharsetTest` / `DatabaseTasksCollationTest` names are
  preserved verbatim in `database-tasks.test.ts`.
