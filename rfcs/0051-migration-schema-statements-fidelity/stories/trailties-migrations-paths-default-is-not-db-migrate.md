---
title: "trailties defaults migration paths to db/migrations, not Rails' db/migrate"
status: in-progress
updated: 2026-08-24
rfc: "0051-migration-schema-statements-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 120
priority: null
pr: 6977
claim: "2026-08-24T11:45:44Z"
assignee: "boot-dump-fingerprint-misses-mysql-functional-index"
blocked-by: null
closed-reason: null
---

## Context

trailties defaults a config's migration discovery paths to `db/migrations`
(primary) and `db/migrations_<name>` (named DBs) —
`migrationsDirsForConfig` in `packages/trailties/src/commands/db.ts:87-97`,
which PR #6974 now resolves into the configuration hash each `HashConfig` is
built from.

Rails' directory is `db/migrate`, everywhere:

- `ActiveRecord::Migrator.migrations_paths = ["db/migrate"]`
  (`vendor/rails/activerecord/lib/active_record/migration.rb:1419`) — the
  fallback every pool uses when its config names none
  (`connection_adapters/abstract/connection_pool.rb:299`).
- `DatabaseTasks.migrations_paths` is
  `Rails.application.paths["db/migrate"].to_a`
  (`lib/active_record/tasks/database_tasks.rb:87-89`), which `db:load_config`
  copies onto `Migrator.migrations_paths`
  (`lib/active_record/railties/databases.rake:27`).

trails already matches on the ActiveRecord side —
`DatabaseTasks.migrationsPaths` and `Migrator.migrationsPaths` both default to
`["db/migrate"]` — so only the trailties spelling diverges, and it diverges
twice: the directory name, and the `_<name>` suffix convention. Rails has no
per-name default at all; a named database gets its paths from the
`migrations_paths` key in `database.yml`
(`database_configurations/hash_config.rb:50-53`) or falls back to the same
`db/migrate`.

## Converged shape

`migrationsDirsForConfig` keeps only the explicit `migrationsPaths` arm and
falls back to `Migrator.migrationsPaths` — i.e. `db/migrate` — for every
config, primary or named. Scaffolding (`trails generate migration` and
friends) writes to the same directory. Any existing `db/migrations` handling is
removed rather than aliased.

## Acceptance criteria

- [ ] `migrationsDirsForConfig` no longer synthesizes `db/migrations` or
      `db/migrations_<name>`; the fallback is `Migrator.migrationsPaths`.
- [ ] Migration scaffolding writes into `db/migrate`.
- [ ] `packages/trailties/src/commands/db.test.ts` keeps its test names and
      passes against the new default.
