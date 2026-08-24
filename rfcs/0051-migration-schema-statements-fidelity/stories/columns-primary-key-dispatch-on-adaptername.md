---
title: "columns/primary_key dispatch on adapterName instead of per-adapter overrides"
status: ready
updated: 2026-08-24
rfc: "0051-migration-schema-statements-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 300
priority: 51
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

# `columns` / `primary_key` dispatch on `adapterName` instead of per-adapter overrides

## Context

Surfaced converging RFC 0106's call-set rows for
`connection-adapters/abstract/schema-statements.ts` in PR #6560. Three rows
(`columns` → `column_definitions`, `columns` → `new_column_from_field`,
`primary_key` → `primary_keys`) could not be converged and were left baselined
with a reviewed reason, because the calls have nowhere to land.

Rails puts one generic body in the abstract mixin and lets each adapter override
the primitive it calls:

`vendor/rails/activerecord/lib/active_record/connection_adapters/abstract/schema_statements.rb:107-113`

```ruby
def columns(table_name)
  table_name = table_name.to_s
  definitions = column_definitions(table_name)
  definitions.map do |field|
    new_column_from_field(table_name, field, definitions)
  end
end
```

`schema_statements.rb:145-149`

```ruby
def primary_key(table_name)
  pk = primary_keys(table_name)
  pk = pk.first unless pk.size > 1
  pk
end
```

`column_definitions`, `new_column_from_field` and `primary_keys` are then
defined per adapter (`sqlite3/schema_statements.rb`,
`postgresql/schema_statements.rb`, `mysql/schema_statements.rb`).

trails instead writes both bodies as one `switch (this.adapterName)` with a
`case "sqlite"` / `case "postgres"` / `case "mysql"` arm inlining each adapter's
PRAGMA or catalog query
(`packages/activerecord/src/connection-adapters/abstract/schema-statements.ts`,
`columns` and `primaryKey`). The generic Rails body does not exist, so there is
no `column_definitions` / `new_column_from_field` / `primary_keys` seam to call.

RFC 0026 (adapter-layout-fidelity) did this extraction for several other PG
schema-statement groups but is now closed with these two still un-extracted.

## Converged shape

- `SchemaStatements#columns` becomes the Rails three-liner: `columnDefinitions`
  then `map` to `newColumnFromField`.
- `SchemaStatements#primaryKey` becomes `primaryKeys(tableName)`, collapsing to
  the single value unless composite.
- Each `switch` arm moves to the matching adapter's own schema-statements mixin
  as `columnDefinitions` / `newColumnFromField` / `primaryKeys`.

## Acceptance criteria

- [ ] No `adapterName` switch remains in `columns` or `primaryKey`; each adapter
      carries its own primitive at the Rails name and Rails file path.
- [ ] The three rows are deleted by hand from
      `scripts/api-compare/call-mismatches-exclude/activerecord/connection-adapters/abstract/schema-statements.json`
      and the mark tightened with `pnpm parity:api:calls:tighten`. No reseed.
- [ ] No `current_adapter?` arm is dropped — every existing switch arm survives
      as an override.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.
