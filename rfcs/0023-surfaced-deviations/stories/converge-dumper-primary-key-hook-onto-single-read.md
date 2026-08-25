---
title: "Converge the dumper's resolvePrimaryKeyColumns hook tree onto Rails' single primary_key(table) read"
status: draft
updated: 2026-08-05
rfc: "0023-surfaced-deviations"
cluster: null
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

Surfaced by #6113 (`converge-pg-column-definitions-drop-indisprimary`). That PR
dropped PG's trails-only `indisprimary` column-definitions field by giving the
PG dumper a `resolvePrimaryKeyColumns` override, matching the MySQL dumper's
existing one. That fixed the reflection-layer deviation but left the dumper-layer
one live, and now triplicated.

Rails' `SchemaDumper#table`
(`vendor/rails/activerecord/lib/active_record/schema_dumper.rb:160-186`) resolves
the primary key in one line, once, before it looks at any column:

```ruby
pk = @connection.primary_key(table)
...
case pk
when String
  tbl.print ", primary_key: #{pk.inspect}" unless pk == "id"
  pkcol = columns.detect { |c| c.name == pk }
```

There is no per-column primary flag and no dialect hook: `primary_key(table)` is
authoritative for every adapter, and the dumper `detect`s the column by name.

trails instead has a three-method hook tree that Rails has no counterpart for:

- `SchemaDumper#resolvePrimaryKeyColumns` (`schema-dumper.ts:834`) — base
  implementation filtering on the per-column `primaryKey` flag, then reordering,
- `SchemaDumper#orderPrimaryKeyColumns` (`schema-dumper.ts:847`) — the reorder,
- dialect overrides in `mysql/schema-dumper.ts:65` and (as of #6113)
  `postgresql/schema-dumper.ts`, both of which exist purely to _ignore_ the
  per-column flag and read `primaryKeyOrderCache` instead.

Two of three adapters override the base method to not use it, which is the
signal that the base method is the wrong shape. `primaryKeyOrderCache`
(`schema-dumper.ts:374`, populated in `table()` from `adapter.primaryKeys`) is
already the `@connection.primary_key(table)` analogue.

## Converged shape

Resolve the PK from `primaryKeyOrderCache` unconditionally in the one place
Rails does, and delete the hook tree:

- base `resolvePrimaryKeyColumns` reads the cache and `detect`s columns by name,
  with no per-column-flag filter,
- both dialect overrides are deleted (they become the base behaviour),
- `orderPrimaryKeyColumns` folds into that single read if it has no other caller.

The per-column `Column.primaryKey` flag then has no dumper consumer at all,
which is the precondition for retiring it from the remaining adapters — see
[[converge-mysql-column-primarykey-flag-promoted-unique]] (done) for the MySQL
reflection half and the schema-cache / introspection fallbacks it lists.

## Acceptance criteria

- [ ] One `resolvePrimaryKeyColumns` implementation; `mysql/schema-dumper.ts`
      and `postgresql/schema-dumper.ts` carry no override.
- [ ] No dumper path reads `ColumnInfo.primaryKey`.
- [ ] Dumps unchanged on all three adapters, including composite-PK tables, PK
      column ordering, and the MySQL promoted-unique case (`string_key_objects`
      dumps `id: false`) — the case the MySQL override was added for.
- [ ] `pnpm parity:api:extra --package activerecord` shows no new surface; the hook
      count goes down.
