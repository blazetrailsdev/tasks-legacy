---
title: "Fold InsertAll's _populateUpdatableColumns deferral back into the Rails constructor"
status: draft
updated: 2026-08-21
rfc: "0023-surfaced-deviations"
cluster: rails-deviation
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 200
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`packages/activerecord/src/insert-all.ts` defers four of Rails'
`InsertAll#initialize` steps out of the constructor into a trails-invented
private `_populateUpdatableColumns()` that every entry point awaits:

- `@unique_by = find_unique_index_for(@unique_by)` — `vendor/rails/activerecord/lib/active_record/insert_all.rb:42`
- `@returning = (connection.supports_insert_returning? ? primary_keys : false) if @returning.nil?` — `insert_all.rb:38-40`
- `primary_keys`' `schema_cache.primary_keys(model.table_name)` read — `insert_all.rb:60-62`
  (memoized into `_schemaCachePrimaryKeys`; `primaryKeys()` falls back to
  `model.primaryKey` until it resolves, and never makes the `table_name` call)
- the `elsif @on_duplicate == :update && updatable_columns.empty?` branch of
  `configure_on_duplicate_update_logic` — `insert_all.rb:140`

Each of the four is async in trails only because the schema-cache reads
(`primary_keys`, `indexes`, `supports_insert_returning?`) are async here.
PR #6841 converged the rest of this file's call set and left the four as
call-site receipts rather than baseline rows: two `@missingRailsCall` tags
(`find_unique_index_for` on the constructor, `table_name` on `primaryKeys()`,
`empty?` on `configureOnDuplicateUpdateLogic`) plus a `@missingRailsArgs
except — PERMANENT` for the receiver-form `except`.

The receipts are debt, not permission: `_populateUpdatableColumns` has no Rails
counterpart, and the split means `returning` / `uniqueBy` / `onDuplicate` are
observably not-yet-final between construction and the first await, which the
`_returningDefaulted` and `_uniqueByResolved` flags exist only to paper over.

## Converged shape

`initialize` runs all four steps in Rails' order, with no deferral method and
no `_returningDefaulted` / `_uniqueByResolved` / `_schemaCachePrimaryKeys`
flags. That requires a synchronous schema-cache read for `primary_keys`,
`indexes`, and `supports_insert_returning?` on the warm-cache path — the same
always-warm posture RFC 0031 took for `columnsHash()`. If a warm-cache sync
read is available, fold `_populateUpdatableColumns()` back into the constructor
and delete the three receipts plus the two flags; `primaryKeys()` then reads
`this.model.schemaCache.primaryKeys(this.model.tableName)` directly, making
Rails' `table_name` call.

## Acceptance criteria

- [ ] `_populateUpdatableColumns()` is gone; `initialize` performs Rails'
      steps in `insert_all.rb:18-45` order.
- [ ] `primaryKeys()` makes the `table_name` call (`insert_all.rb:61`).
- [ ] The `@missingRailsCall find_unique_index_for` / `table_name` / `empty?`
      receipts are deleted, not reworded.
- [ ] `pnpm parity:api:calls`, `parity:api:calls:args`,
      `parity:api:extra --package activerecord` green; no new baseline rows.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.

If a synchronous warm-cache read genuinely cannot be had, `pnpm tasks block`
with the specific blocker — do not close this by rewording the receipts.
