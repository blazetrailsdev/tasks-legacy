---
title: "migrator-pending-migrations-must-not-create-schema-table"
status: claimed
updated: 2026-08-24
rfc: "0051-migration-schema-statements-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: null
claim: "2026-08-24T13:05:17Z"
assignee: "migrator-pending-migrations-must-not-create-schema-table"
blocked-by: null
closed-reason: null
---

## Context

`ActiveRecord::Migrator#pending_migrations`
(`vendor/rails/activerecord/lib/active_record/migration.rb:1475-1478`) is two
lines and touches nothing:

```ruby
def pending_migrations
  already_migrated = migrated
  migrations.reject { |m| already_migrated.include?(m.version) }
end
```

trails' `Migrator#pendingMigrations`
(`packages/activerecord/src/migration.ts:2835-2839`) opens with
`await this._ensureSchemaTable()` — a **write**. Reading which migrations are
pending therefore CREATES `schema_migrations` (and, through
`_ensureSchemaTable`, `ar_internal_metadata`) on a database that has neither.
Rails never creates a table from this reader; `databases.rake:333`
(`pool.migration_context.open.pending_migrations`) and
`Migration.pending_migrations` (`migration.rb:758-769`) both call it expecting a
pure read.

Surfaced in PR #6978, which converged `activerecord-cli`'s
`checkPendingMigrations` onto the `databases.rake:330-336` reader
(`pool.migrationContext.open().pendingMigrations()`). That is the right call
shape, and it inherits this side effect: `ar db:abort_if_pending_migrations`
now creates `schema_migrations` on a fresh database.

The extra call is also what forced the trails-only
`Migrator#pendingMigrationsReadOnly` (`migration.ts:2823-2828`) into existence —
a second spelling of the same Rails method that exists only to avoid the write.
It has no Ruby counterpart and two remaining callers
(`packages/trailties/src/commands/db.ts:840`,
`packages/activerecord/src/migrator.trails.test.ts:205`).

Note the asymmetry that makes this safe to converge: `MigrationContext`'s own
`pending_migration_versions` (`migration.rb:1299-1301`) goes through
`get_all_versions` (`:1282-1288`), which IS `table_exists?`-guarded and returns
`[]` on a miss. `Migrator#migrated` → `load_migrated` (`:1480-1486`) is not, by
design — the two readers answer at different levels.

## Acceptance criteria

- [ ] `Migrator#pendingMigrations` drops `_ensureSchemaTable()` and mirrors
      `migration.rb:1475-1478` exactly.
- [ ] `pendingMigrationsReadOnly` is deleted, with its two callers moved onto
      the Rails-named method (or onto `MigrationContext#pendingMigrationVersions`
      where that is the reader Rails uses at that site).
- [ ] The 17 existing `pendingMigrations()` call sites still pass; any that
      relied on the implicit table creation get the `ensure` Rails puts at
      _their_ level, not inside the reader.
- [ ] `pnpm parity:api:calls` and `pnpm parity:api:calls:args` green — the
      dropped call is a converged row to DELETE from the baseline, not to add.
