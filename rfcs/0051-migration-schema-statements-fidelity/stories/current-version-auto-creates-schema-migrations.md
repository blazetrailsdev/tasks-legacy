---
title: "Migrator#currentVersion auto-creates schema_migrations"
status: closed
updated: 2026-08-24
rfc: "0051-migration-schema-statements-fidelity"
cluster: null
deps: []
deps-rfc: []
est-loc: 90
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: "Delivered by PR #6982 (ef6c834ec 'Migrator keeps only its migration.rb:1404 surface'). currentVersionReadOnly is gone: `git grep currentVersionReadOnly origin/main -- packages` returns nothing. Migrator#currentVersion (migration.ts:2705-2708) is now exactly migration.rb:1435-1437 — `migrated.max || 0` — with no _ensureSchemaTable call. The story's own post-#6981 correction identified deleting currentVersionReadOnly as the only genuinely convergeable part; that is done."
---

## Context

`Migrator#currentVersion` (`packages/activerecord/src/migration.ts:2559`) calls
`_ensureSchemaTable()` before reading, so asking for the current version
**creates** `schema_migrations` as a side effect. Rails' `Migrator#current_version`
(`vendor/rails/activerecord/lib/active_record/migration.rb:1290`) is just
`get_all_versions.max || 0`, and `get_all_versions` returns `[]` when
`schema_migration.table_exists?` is false — it never creates anything.

trails papered over this with a second method, `Migrator#currentVersionReadOnly`
(`migration.ts:2586`), whose own JSDoc says it "Matches Rails' `current_version`
exactly" and that the divergent `currentVersion` "keeps the legacy auto-create
path to stay compatible with internal callers that rely on it". So the faithful
behaviour exists but sits behind a trails-invented name, while the Rails-named
method is the deviant one.

Surfaced while moving the DatabaseTasks environment checks off `Migrator`
(#5861): the deleted `Migrator#lastStoredEnvironment` had to call
`currentVersionReadOnly()` precisely because the Rails-named `currentVersion`
would have created the table during a read-only guard.

Remaining callers of the read-only variant: `tasks/database-tasks.ts:1037` and
`packages/trailties/src/commands/db.ts:716`.

## Correction before claiming (added post-#6981)

**This story cites the wrong `current_version`, and criterion 1 as written would
ADD a divergence.** `vendor/rails/activerecord/lib/active_record/migration.rb`
defines it twice:

- `:1290` — `MigrationContext#current_version` (class opens `:1211`):
  `get_all_versions.max || 0`, and `get_all_versions` (`:1282-1288`) IS
  `table_exists?`-guarded. This is the one the Context text quotes.
- `:1435` — `Migrator#current_version` (class opens `:1405`): `migrated.max || 0`,
  guarded by nothing. `Migrator#initialize` (`:1421-1432`) creates both
  bookkeeping tables after `validate`, so a `Migrator` read DOES create them in
  Rails.

`Migrator#currentVersion` is the `:1435` one. Making it "return 0 when the table
is absent" would diverge from Rails, not converge toward it.

The same premise error produced the sibling story
`migrator-pending-migrations-must-not-create-schema-table`, whose headline
("pending reader must not create the schema table") is likewise not Rails
behavior — `MigrationContext#open` (`:1413-1415`) builds a `Migrator`, whose
constructor creates the tables, so `databases.rake:333`'s
`open.pending_migrations` writes on a fresh database in Rails too. #6981
converged the reader body (`migrated` + `reject`, `migration.rb:1475-1478`) and
deleted `pendingMigrationsReadOnly`, and the reviewer confirmed the remaining
write as justified.

The genuinely convergeable part of this story survives and is worth doing:
**delete `currentVersionReadOnly`** (trails-only spelling of a Rails method) and
move its two callers onto `currentVersion`, or onto
`MigrationContext#currentVersion` where a guarded read is what the site actually
wants — that is the reader Rails uses for a no-write current-version probe.
Criterion 1 should be restated as "`Migrator#currentVersion` mirrors
`migration.rb:1435` exactly (`migrated.max || 0`) with the table creation left
at the constructor stand-in", not as a behavior change.

The lazy-`_ensureSchemaTable` shape itself is already tracked and closed by
[[migrator-defers-schema-table-creation-to-lazy-ensure]] (#5777).

## Acceptance criteria

- [ ] `Migrator#currentVersion` does not create `schema_migrations` — it reads
      through `getAllVersions()` and returns 0 when the table is absent, per
      `migration.rb:1290`.
- [ ] `currentVersionReadOnly` is deleted; its two callers use `currentVersion`.
- [ ] The internal callers that relied on the auto-create side effect call
      `_ensureSchemaTable()` explicitly where they genuinely need the table,
      rather than depending on a reader to create it.
- [ ] Existing migrator/migration-context suites pass unchanged.
