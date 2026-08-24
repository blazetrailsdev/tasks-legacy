---
title: "Wire validate_table_length! into create_table"
status: closed
updated: 2026-08-24
rfc: "0051-migration-schema-statements-fidelity"
cluster: null
deps: []
deps-rfc: []
est-loc: 40
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: "Delivered. abstract/schema-statements.ts createTable now runs both checks at Rails' position, before the force/ifNotExists handling: this.validateCreateTableOptionsBang(options) at :398 and this.validateTableLengthBang(tableName) at :401, mirroring schema_statements.rb:294-295 — the force+ifNotExists raise follows at :404. (The story cited :245 for the unwired body; the call site is now :398-401.)"
---

## Context

Surfaced while reviewing PR #5574 (`port-migration-rename-table-cases`).

That PR wired `validateTableLengthBang` into `renameTable` on all three
adapters, which was its only Rails call site relevant to the ported tests. But
Rails calls `validate_table_length!` from **two** places, and the other one is
still unwired:

`vendor/rails/activerecord/lib/active_record/connection_adapters/abstract/schema_statements.rb:295`

```ruby
def create_table(table_name, id: :primary_key, primary_key: nil, force: nil, **options, &block)
  validate_create_table_options!(options)
  validate_table_length!(table_name) unless options[:_uses_legacy_table_name]
```

`SchemaStatements#createTable` in
`packages/activerecord/src/connection-adapters/abstract/schema-statements.ts:245`
performs neither check — it goes straight to the `force`/`ifNotExists` handling.
So `createTable` with an over-long name does not raise `ArgumentError` the way
Rails does; it reaches the database and fails (or truncates) per-adapter.

`validateTableLengthBang` already exists at
`connection-adapters/abstract/schema-statements.ts:2560` and needs no new code —
only the call site.

Note the `_uses_legacy_table_name` guard is deliberately NOT part of this work:
its only producer in Rails is `Migration::Compatibility::V7_0`, which trails
does not implement (PR #5070 closed unmerged by product decision). Call
`validateTableLengthBang` unconditionally, as PR #5574 did for `renameTable`.

Sibling story: `wire-sqlite-validate-index-length-override` (same
validate\_\*\_length family, different override).

## Acceptance criteria

- [ ] `createTable` calls `validateTableLengthBang(tableName)` at the position
      Rails calls it (before the force/if_not_exists checks).
- [ ] A test covers the over-long-name `ArgumentError`, with the Rails message
      `Table name '<name>' is too long; the limit is <N> characters`.
- [ ] Green on sqlite3, PostgreSQL, and MySQL.
- [ ] `parity:api` / `parity:test` delta non-negative; if the wide call
      ratchet reports STALE entries for `create_table` → `validate_table_length!`,
      remove exactly those by hand (never `--write`).
