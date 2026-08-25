---
title: "trailties db migrate re-derives pending counts in a second pool for non-Rails messages"
status: draft
updated: 2026-08-02
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "trailties"
deps: []
deps-rfc: []
est-loc: 70
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`db:migrate` in Rails is `DatabaseTasks.migrate_all` followed by `db:_dump`
(`activerecord/lib/active_record/railties/databases.rake:88-92`) — it prints
nothing beyond what `Migration#write` emits.

trailties' `db migrate` adds two messages that have no Rails source:
"No migrations found." when no config has migration files, and
"All migrations are up to date." when a config ends with zero pending
(`packages/trailties/src/commands/db.ts`). Keeping the second one across the
`migrateAll` delegation added in PR #5864 costs a whole extra pass after the
migration run: for every config it opens another temporary pool, builds a
throwaway `Migrator` over the already-discovered migrations, and calls
`pendingMigrations()` purely to decide whether to print. That is one extra
pool plus one `schema_migrations` read per database on every `db migrate`.

Either drop both messages as trails inventions, or keep them and source the
pending count from the run that just happened rather than re-deriving it.

## Acceptance criteria

- [ ] The post-`migrateAll` loop no longer opens a second pool per config
      solely to compute a pending count for the message.
- [ ] "No migrations found." / "All migrations are up to date." are either
      removed as trails inventions or justified at the call site against
      `databases.rake`.
- [ ] The `dumpSchemaAfterMigration`-gated dump still runs inside the pool
      that performed the migration — a `:memory:` database loses its data
      when its pool is replaced, which is what
      "db schema:dump --format=sql works against ':memory:' sqlite by reusing
      the migration adapter" in `packages/trailties/src/commands/db.test.ts`
      pins.
