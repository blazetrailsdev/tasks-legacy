---
title: "migration-migrate-drops-pool-with-connection"
status: done
updated: 2026-08-24
rfc: "0051-migration-schema-statements-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: 6971
claim: "2026-08-24T03:57:41Z"
assignee: "migration-migrate-drops-pool-with-connection"
blocked-by: null
closed-reason: null
---

## Context

`ActiveRecord::Migration#migrate` (`vendor/rails/activerecord/lib/active_record/migration.rb:968-984`)
wraps the whole `exec_migration` call in a checkout:

```ruby
ActiveRecord::Tasks::DatabaseTasks.migration_connection.pool.with_connection do |conn|
  time_elapsed = ActiveSupport::Benchmark.realtime do
    exec_migration(conn, direction)
  end
end
```

trails' `Migration#migrate`
(`packages/activerecord/src/migration.ts:1285-1292`) instead runs
`await this.execMigration(this.connection, direction)` against the
already-seated `this.connection`, so it never reaches
`DatabaseTasks.migrationConnection`, its pool, or `withConnection`. The
connection the migration body sees is therefore whatever was seated on the
instance rather than one checked out for the duration of the migration.

This is the last remaining `kind: "set"` divergence recorded for
`migration.ts` `migrate` (surfaced by RFC 0106's
`wave-5c-migration-shard-sweep`, which migrated it to a `@missingRailsCall`
tag at the call site). It converges with RFC 0073's permanent-checkout flip:
once a migration can check a connection out per run, `migrate` should seat
`conn` from `DatabaseTasks.migrationConnection().pool().withConnection(...)`
exactly as Rails does.

## Acceptance criteria

- [ ] `Migration#migrate` obtains its connection from
      `DatabaseTasks.migrationConnection`'s pool via `withConnection`, and
      passes that connection to `execMigration`, mirroring
      `migration.rb:973-978`.
- [ ] The `@missingRailsCall with_connection` tag on `Migration#migrate` in
      `packages/activerecord/src/migration.ts` is deleted.
- [ ] `pnpm parity:api:calls` and `pnpm parity:api:calls:args` green.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.
