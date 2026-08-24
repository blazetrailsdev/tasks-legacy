---
title: "migrator-run-surface-caller-migration"
status: in-progress
updated: 2026-08-24
rfc: "0051-migration-schema-statements-fidelity"
cluster: null
deps:
  - unify-migration-discovery-delete-registered-migrations-seam
deps-rfc: []
est-loc: null
pr: 6982
claim: "2026-08-24T13:19:12Z"
assignee: "migrator-run-surface-caller-migration"
blocked-by: null
closed-reason: null
---

## Context

`migrator-keeps-only-its-rails-1404-surface` finished the first half: every
`MigrationContext` run method in `packages/activerecord/src/migration.ts` now
has a real body (`migrate` / `up` / `down` / `run` / `migrationsStatus` /
`move` / `lastStoredEnvironment` / `protectedEnvironment`), so nothing on
`MigrationContext` delegates back into `Migrator` any more, and the discovery
statics (`Migrator.fromPath` / `fromDir` / `fromPaths` / `discoverMigrations`)
plus `Migrator#migrationsPaths`, `Migrator#isProtectedEnvironment` and
`Migrator#move` are gone.

What is left under the banner
`// --- MigrationContext-style methods (Rails: MigrationContext) ---` in
`Migrator` is the set whose _callers_ still hold a `Migrator`:
`schemaMigration`, `open`, `needsMigration`, `pendingMigrationVersions`,
`currentEnvironment`, `lastStoredEnvironment`, plus the MigrationContext-shaped
`migrate(targetVersion, block)` / `up` / `down` / `rollback` / `forward` /
`run(direction, target)` / `migrationsStatus` / `pendingMigrations`. Rails'
`Migrator` (`vendor/rails/activerecord/lib/active_record/migration.rb:1404-1620`)
owns only `current_version` / `current_migration` (alias `current`) / `run` /
`migrate` — both no-arg, reading `@direction` / `@target_version` —
`runnable` / `migrations` / `pending_migrations` / `migrated` / `load_migrated`
and the privates, plus the statics `migrations_paths` + `current_version`.

The blocker is the ~24 non-test call sites that construct
`new Migrator(adapter, migrations, options)` from an in-memory
`MigrationProxy[]` and then call the MigrationContext-shaped surface:
`packages/trailties/src/commands/db.ts` (`createMigrator` /
`withMigratorForDb` and ~10 command bodies),
`packages/activerecord/src/tasks/database-tasks.ts:343,619,1148`,
`packages/activerecord-cli/src/pending-migrations.ts`,
`packages/activerecord/src/test-databases.ts`,
`packages/website/src/lib/frontiers/trail-cli.ts`, plus `CheckPending`.
Rails' `MigrationContext` is path-based (`migrations_paths`, constructor arg),
so repointing those callers needs a decision about how a caller holding a
pre-built migration list reaches a `MigrationContext` — trailties has its own
`discoverMigrations` loader (`packages/trailties/src/migration-loader.ts`)
rather than `MigrationContext#migrations`.

`MigrationContext#up` / `down` / `run` also spell out
`isUseAdvisoryLock() ? withAdvisoryLock(...) : ...` inline because
`Migrator#migrate` / `#run` still carry the MigrationContext-shaped signatures;
Rails just writes `Migrator.new(...).migrate` / `.run`.

## Decision: how a caller holding a pre-built migration list reaches a MigrationContext

**Superseded, 2026-08-08.** The previous text here specified a fourth optional
`registeredMigrations?: MigrationProxy[]` constructor argument on
`MigrationContext`, settled by
`migration-context-built-by-subclass-override-not-paths`. **That is dead:** PR
5860 was closed unmerged, and the argument does not exist — verified on
`origin/main`, `git grep registeredMigrations packages/activerecord/src`
returns no hits. Do not implement against it.

The question is now settled instead by
`unify-migration-discovery-delete-registered-migrations-seam` (this story's
declared dep, currently `ready`), which **unifies the two discovery paths**
rather than adding a seam for pre-built lists: trailties' `migration-loader`
and `MigrationContext#migrations` stop being two ways to reach the same thing.
Read that story's body first and take the shape it lands as given; the callers
listed above then reach the run surface through a `MigrationContext` built the
unified way, with no `registeredMigrations` argument anywhere.

## Related, and worth landing first

`migrator-still-carries-up-down-rollback-forward` (now `ready`) removes
`Migrator#up` / `#down` / `#rollback` / `#forward` specifically. Verified on
`origin/main`: only **two** non-test callers reach those —
`test-databases.ts:33` (`migrator.up()`) and
`trail-cli.ts:232` (`migrator.rollback(step)`) — so it is landable
independently and shrinks this story's remaining surface. Prefer taking it
first.

## Acceptance criteria

- [ ] The `MigrationContext-style` banner block is gone from `Migrator`.
- [ ] `Migrator#migrate` and `Migrator#run` take no arguments and read
      `@direction` / `@target_version`, as `migration.rb:1444-1458` does.
- [ ] `MigrationContext#up` / `down` / `run` call `Migrator#migrate` /
      `Migrator#run` directly instead of spelling out the advisory-lock pair.
- [ ] Every caller listed above reaches the run surface through a
      `MigrationContext`, built the way
      `unify-migration-discovery-delete-registered-migrations-seam` landed —
      with no `registeredMigrations` constructor argument.
- [ ] `Migrator` keeps only what `migration.rb:1404-1620` gives it.
- [ ] Existing migrator / migration / trailties `db` tests keep their
      Rails-verbatim names and pass.

Hard rules: no `node:*` imports, no `process.*`, async fs only, no new runtime
deps, the LOC ceiling, single PR from main.
