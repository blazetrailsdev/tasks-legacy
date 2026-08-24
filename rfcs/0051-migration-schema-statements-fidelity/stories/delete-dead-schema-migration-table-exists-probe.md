---
title: "Delete Migrator#schemaMigrationTableExists, dead trails-only surface"
status: done
updated: 2026-08-24
rfc: "0051-migration-schema-statements-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 30
priority: null
pr: 6982
claim: "2026-08-24T14:48:31Z"
assignee: "converge-numericality-bigint-exponent-skip"
blocked-by: null
closed-reason: null
---

## Context

`Migrator#schemaMigrationTableExists` (`packages/activerecord/src/migration.ts:2789`)
is trails-only surface on a Rails-matched class. Rails' `Migrator` has no such
method; the only `table_exists?` probe on this path is the one INSIDE
`MigrationContext#get_all_versions`
(`vendor/rails/activerecord/lib/active_record/migration.rb:1282-1288`), which is
a private detail of that reader, not a public predicate a caller consults first.

It is now **dead**: its only caller was `db prepare`'s fresh-vs-initialized
probe, removed when #6978 dropped the preflight migration scans. `grep -rn
schemaMigrationTableExists packages/ --include=*.ts` returns the definition and
the built `.d.ts`, nothing else.

Surfaced in #6981, which deleted its sibling `Migrator#pendingMigrationsReadOnly`
— the same pattern of a trails-invented read-only accessor added to work around
`_ensureSchemaTable`. `Migrator#currentVersionReadOnly` is the third, tracked by
[[current-version-auto-creates-schema-migrations]].

## Converged shape

Delete the method. A caller that genuinely needs the fresh-vs-initialized
distinction asks it the way Rails does — through `MigrationContext`'s
`get_all_versions`-backed readers (`pendingMigrationVersions`, `currentVersion`),
which return `[]`/0 on a missing table rather than exposing a bare predicate.

## Acceptance criteria

- [ ] `Migrator#schemaMigrationTableExists` is deleted, with no replacement
      predicate added elsewhere on `Migrator`.
- [ ] `pnpm parity:api:extra --package activerecord` shows one fewer novel
      public name on `migration.ts`; no baseline row is added.
- [ ] Existing migrator / migration-context / trailties db suites pass unchanged.
