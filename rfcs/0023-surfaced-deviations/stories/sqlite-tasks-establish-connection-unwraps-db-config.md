---
title: "SQLiteDatabaseTasks#establishConnection unwraps DatabaseConfig where Rails passes the object"
status: draft
updated: 2026-08-14
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

Surfaced in PR #6540 (`naming-burndown-3-ar-model-encryption-tasks`) as the last
call-argument `naming` row in `tasks/`.

Rails `activerecord/lib/active_record/tasks/sqlite_database_tasks.rb:72-74`:

```ruby
def establish_connection(config = db_config)
  ActiveRecord::Base.establish_connection(config)
end
```

Rails hands `Base.establish_connection` the **DatabaseConfig object itself**.
`Base.establish_connection` accepts a config object, a symbol, or a hash
(`connection_handling.rb#establish_connection`), and passing the object is what
lets it keep the config's identity — its `name`, `env_name`, and adapter
resolution.

trails `packages/activerecord/src/tasks/sqlite-database-tasks.ts:317-321`:

```ts
private async establishConnection(config: DatabaseConfig = this.dbConfig): Promise<void> {
  await Base.establishConnection(
    config.configuration as { adapter?: string; [key: string]: unknown },
  );
  await (await this.connection()).connectBang();
}
```

It unwraps to the raw `config.configuration` hash and casts, so the
DatabaseConfig identity is discarded before `establishConnection` sees it. That
is an a3 (a conversion Rails does not perform), not a rename — the recorder flags
it as `ref:configuration` vs Rails' `ref:config`.

The likely reason is that trails' `Base.establishConnection` does not accept a
`DatabaseConfig` object; confirm that before converging, since the fix may be to
widen `establishConnection`'s accepted shapes rather than to change this caller.

## Converged shape

```ts
await Base.establishConnection(config);
```

with `Base.establishConnection` accepting a `DatabaseConfig` as Rails does. The
trailing `connectBang()` is a separately-documented deviation and stays.

## Acceptance criteria

- [ ] `establishConnection` passes the config object, not `config.configuration`.
- [ ] The `as { adapter?: string; ... }` cast is gone.
- [ ] `Base.establishConnection` accepts a DatabaseConfig if it did not already.
- [ ] The `tasks/sqlite-database-tasks.ts` `naming` call-arg row is gone.
