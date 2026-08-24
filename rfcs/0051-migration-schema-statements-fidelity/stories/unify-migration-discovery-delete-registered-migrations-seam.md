---
title: "Unify the two migration discovery paths and delete the registeredMigrations seam"
status: in-progress
updated: 2026-08-24
rfc: "0051-migration-schema-statements-fidelity"
cluster: null
deps: []
deps-rfc: []
est-loc: null
pr: 6974
claim: "2026-08-24T09:03:41Z"
assignee: "unify-migration-discovery-delete-registered-migrations-seam"
blocked-by: null
closed-reason: null
---

## Context

`MigrationContext#migrations`
(`vendor/rails/activerecord/lib/active_record/migration.rb:1303-1315`) has
exactly one source: `migration_files` scanning `migrations_paths`, and the
constructor at `migration.rb:1214` takes exactly three arguments. trails has
two migration sources, and PR #5860 made the second one explicit rather than
removing it: `MigrationContext`'s constructor now takes a fourth optional
argument, `registeredMigrations?: MigrationProxy[]`
(`packages/activerecord/src/migration.ts:1729-1757`), and `#migrations`
returns it when present (`migration.ts:1988`). That replaced the anonymous
`MigrationContext` subclass `DatabaseTasks._migrationContextFor` used to build
— Rails' test-only `migrator_class` override (`test/cases/migrator_test.rb`)
sitting in production code — so the seam is strictly better than what it
replaced, but it is still surface Rails does not have.

The root deviation is that trails discovers migrations twice:

- `MigrationContext#migrations` / `migrationFiles`
  (`packages/activerecord/src/migration.ts:1986-2031`) scans `migrationsPaths`
  for `/^\d+_.*\.(ts|js)$/`.
- `packages/trailties/src/migration-loader.ts` is a separate loader (hyphen
  aliases and other spellings `parseMigrationFilename` does not accept), whose
  output is handed to `DatabaseTasks.registerMigrations`
  (`packages/activerecord/src/tasks/database-tasks.ts:314-324`) and read back
  by `_migrationsFor` (`database-tasks.ts:333-338`).

The two are not interchangeable today, which is why the in-memory list has to
reach `MigrationContext` at all.

## Acceptance criteria

- [ ] The two discovery paths are unified: either `migration-loader`'s extra
      filename spellings move into `MigrationContext#parseMigrationFilename` /
      `migrationFiles` and trailties calls `MigrationContext#migrations`, or
      the loader writes only into paths `MigrationContext` already scans.
      Whichever way, document the filename spellings that survive.
- [ ] `DatabaseTasks.registerMigrations` / `_migrationsFor` /
      `_migrationsByConfig` are gone, or reduced to a path registration.
- [ ] The `registeredMigrations` constructor argument is deleted from
      `MigrationContext`, restoring Rails' three-argument constructor
      (`migration.rb:1214`), and `#migrations` has a single source again.
- [ ] The DEVIATION JSDoc block on the constructor goes with it.
- [ ] trailties `db` command tests and
      `packages/activerecord/src/migration-context.trails.test.ts` keep their
      names and pass; the registered-list test in the latter is deleted along
      with the seam.

Hard rules: no `node:*` imports, no `process.*`, async fs only, no new runtime
deps, the LOC ceiling, single PR from main.

## Progress (2026-08-21)

The **discovery half** landed: `packages/trailties/src/migration-loader.ts` and
its test are deleted, its two extra filename spellings (the pre-1.12c hyphen
alias, and the `.ts`-over-`.js` / underscore-over-hyphen dedupe) moved into
`MigrationContext#migrationFiles` / `#parseMigrationFilename` where they are
documented on `migrationFiles`, and every caller — trailties `db.ts`,
`db.test.ts`, activerecord-cli `db-helpers.ts` — now discovers through
`MigrationContext#migrations`. The four spelling cases moved to
`packages/activerecord/src/migration-context.trails.test.ts` verbatim, and
`db.test.ts`'s anonymous `MigrationContext` subclass is gone with them.

Also already true before this story was written, so its ACs need no work:
`MigrationContext`'s constructor is Rails' three-argument one
(`migration.rb:1214`) — the `registeredMigrations` fourth argument and its
DEVIATION JSDoc are gone, `#migrations` has one source, and there is no
registered-list test to delete.

**What remains** is AC2 alone: `DatabaseTasks.registerMigrations` /
`_migrationsFor` / `_migrationsByConfig` (`tasks/database-tasks.ts:316-354`)
and the anonymous `MigrationContext` subclass in `_migrationContextFor`
(`database-tasks.ts:361-380`). Removing it means the AR tests that register
in-memory `MigrationProxy` objects with closure side effects
(`tasks/database-tasks-rollback.trails.test.ts`,
`tasks/database-tasks-migrate-all-metadata.trails.test.ts`,
`tasks/database-tasks.test.ts:982-1218`, plus
`activerecord-cli/src/__e2e__/helpers.ts`) have to move onto real migration
files under a per-config `migrationsPaths`, which did not fit the PR ceiling
alongside the discovery half. The seam is now a pure transport for an already
unified list, so nothing else blocks it.
