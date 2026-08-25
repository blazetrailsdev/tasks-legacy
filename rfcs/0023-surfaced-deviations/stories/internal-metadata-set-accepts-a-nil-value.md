---
title: "InternalMetadata#set accepts nil so record_environment needs no cast"
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

Rails' `InternalMetadata#[]=` (`activerecord/lib/active_record/internal_metadata.rb`)
takes any value, `nil` included, and `Migrator#record_environment`
(`activerecord/lib/active_record/migration.rb:1512-1516`) writes
`connection.pool.db_config.env_name` straight into it. On a `NullPool` that
reader is `nil` — `NullConfig#method_missing` answers nil for every key
(`activerecord/lib/active_record/connection_adapters/abstract/connection_pool.rb:17-22`),
mirrored by trails at `connection-pool.ts:62-69`.

trails types `InternalMetadata.set(key: string, value: string)`
(`packages/activerecord/src/internal-metadata.ts:167`), so PR #6197's converged
`record_environment` had to write

```ts
this.connection.pool.dbConfig.envName as string;
```

(`packages/activerecord/src/migration.ts`, in `recordEnvironment`). The cast is
inert at runtime — it narrows `string | null | undefined` — but it hides the
`nil` arm from the type system, and a bare-adapter Migrator would stamp
`undefined` where Rails stamps NULL.

## Converged shape

Widen `set` (and the `updateOrCreateEntry` / `updateEntry` / `createEntry`
chain it feeds) to Rails' value domain so a `nil` env name is written as SQL
NULL rather than coerced, then delete the cast in `recordEnvironment`. Check
what `get` should answer for a stored NULL — it already returns `null` for a
null column value (`internal-metadata.ts:155-165`), so the round trip is
consistent once the writer accepts it.

## Acceptance criteria

- [ ] `InternalMetadata#set` accepts the `nil` a `NullConfig#env_name` yields
      and writes NULL.
- [ ] The `as string` cast in `Migrator#recordEnvironment` is DELETED along with
      its comment.
- [ ] A test covers a bare-adapter (NullPool) Migrator's `record_environment`
      stamping NULL rather than the string "undefined".
