---
title: "sqlite3-primary-key-singular-override-has-no-rails-counterpart"
status: draft
updated: 2026-08-25
rfc: "0023-surfaced-deviations"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced in review of #7044 (`sqlite-attached-schema-notion-has-no-rails-counterpart`),
which touched this body while unqualifying its PRAGMA but left the larger
divergence in place as out of scope.

`SQLite3Adapter#primaryKey`
(`packages/activerecord/src/connection-adapters/sqlite3-adapter.ts:2072`) is a
bespoke singular override with **no counterpart in Rails' sqlite3 adapter at
all**. Rails' `sqlite3_adapter.rb` defines only `primary_keys` (plural,
`:281-284`); the singular reader is the abstract three-line delegation:

```ruby
# abstract/schema_statements.rb:145-149
def primary_key(table_name)
  pk = primary_keys(table_name)
  pk = pk.first unless pk.size > 1
  pk
end
```

trails already mirrors both halves faithfully and independently:

- `SchemaStatements#primaryKey`
  (`abstract/schema-statements.ts:1205-1212`) is a correct port of that
  delegation.
- `SQLite3Adapter#primaryKeys`
  (`sqlite3-adapter.ts:1584-1587`) is a correct port of `:281-284`, reading
  through `tableStructure`.

So the override is pure duplication that shadows a working delegation, and it
re-derives the answer down a _different_ reflection path — a direct
`PRAGMA table_info(...)` rather than `tableStructure`.

## Converged shape

Delete `SQLite3Adapter#primaryKey` and let the abstract delegation run through
`primaryKeys`, as it does in Rails.

**Verify the reflection-path swap before deleting** — this is the part that
needs care, not the deletion itself. `tableStructure` selects
`PRAGMA table_xinfo` when `supportsVirtualColumns()` is true, where the deleted
override hard-codes `table_info`. `table_xinfo` returns hidden/generated
columns that `table_info` omits, so confirm a generated-column table still
reports the same primary key both ways before and after.

Also confirm the empty-table arm still agrees: the override returns `null`
explicitly for a rowid-only table, and the delegation reaches the same answer
via `primaryKeys[0] ?? null`. Rails gets `nil` from `pk.first` on an empty
array, so both spellings match Ruby — but pin it with a test rather than
assuming.

## Acceptance criteria

- [ ] `SQLite3Adapter#primaryKey` is gone; the singular reader resolves through
      `SchemaStatements#primaryKey` -> `SQLite3Adapter#primaryKeys`.
- [ ] Scalar, composite, and rowid-only (no explicit PK) tables all return what
      they returned before — `string`, `string[]`, and `null` respectively.
- [ ] A generated/virtual-column table is covered, pinning the
      `table_xinfo` vs `table_info` path swap described above.
- [ ] `pnpm parity:api:extra --package activerecord` novel count does not
      increase (it should DROP by one if the override was counted).
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.
