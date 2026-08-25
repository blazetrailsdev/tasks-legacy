---
title: "MigrationProxy carries Rails' delegated migrate/disable_ddl_transaction members"
status: draft
updated: 2026-08-13
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 250
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Rails' `MigrationProxy` delegates `migrate`, `announce`, `write` and
`disable_ddl_transaction` to the lazily loaded migration
(`vendor/rails/activerecord/lib/active_record/migration.rb:1187`, over
`@migration ||= load_migration` at `:1190-1192`), so `Migrator` reads them off
the proxy directly — `ddl_transaction(migration)` / `use_transaction?(migration)`
(`:1584-1593`) and `migration.migrate(@direction)` (`:1536`).

trails' `MigrationProxy` interface (`packages/activerecord/src/migration.ts`)
exposes only `migration()`, whose loader is an async ESM `import()` where Ruby's
`load` is synchronous. PR #6474 converged the call boundary — the proxy is now
what `executeMigrationInTransaction` passes to `ddlTransaction` and
`isUseTransaction` — but the delegated reads still `await migration.migration()`
at the call site rather than going through proxy members, justified there.

The `deprecator.ts` `MigrationProxy` class already has the delegating members
(`migrate`, `announce`, `write`, and a `disableDdlTransaction` getter that
throws if the migration is not yet loaded); the `migration.ts` interface does
not, and ~45 object-literal proxies in the AR test suite implement that
interface, which is why the delegation was not added in #6474.

## Acceptance criteria

- [ ] The `migration.ts` `MigrationProxy` carries Rails' delegated members, so
      `Migrator` reads `migration.migrate(...)` and the `disableDdlTransaction`
      arm of `isUseTransaction` off the proxy rather than resolving it at the
      call site.
- [ ] The two proxy construction sites in `migration.ts` and the test literals
      are migrated to whatever shape that requires (a class mirroring
      `MigrationProxy = Struct.new(...)` is the Rails shape).
- [ ] The async-load justification comments in `executeMigrationInTransaction`
      and `isUseTransaction` are deleted, or reduced to whatever genuinely
      remains language-forced.
- [ ] No baseline row added; migrator/migration suites pass on all three
      adapters.
