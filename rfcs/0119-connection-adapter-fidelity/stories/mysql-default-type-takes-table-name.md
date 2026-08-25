---
title: "mysql-default-type-takes-table-name"
status: ready
updated: 2026-08-25
rfc: "0119-connection-adapter-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 50
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while burning down RFC 0096 wave-2 naming rows (PR #6433).

`MySQL::SchemaStatements#new_column_from_field`
(`vendor/rails/activerecord/lib/active_record/connection_adapters/mysql/schema_statements.rb:189-206`)
calls `default_type(table_name, field_name)` — `default_type` takes the
**table name** and looks up the create-table info itself:

```ruby
elsif default && default_type(table_name, field_name) == :function
```

trails (`packages/activerecord/src/connection-adapters/mysql/schema-statements.ts:345-347,394`)
declares `defaultType(createTableInfo: string | null, fieldName: string)` and
the caller hoists the lookup into the argument —
`defaultType(createTableInfoFn(tableName), fieldName)`. The signature diverges,
which is why the RFC 0096 row (Ruby `ref:tableName, ref:fieldName` → TS
`ref:createTableInfoFn, ref:fieldName`) is an a3 finding rather than a rename.

Converged shape: `defaultType(tableName, fieldName)` with the create-table-info
lookup inside the method, as in Rails; the caller passes `tableName`.

Related but distinct and already done:
[[sqlite-columns-converge-new-column-from-field]] (SQLite3 adapter).

## Acceptance criteria

- [ ] `defaultType` takes `(tableName, fieldName)` and resolves the create-table
      info internally, mirroring `mysql/schema_statements.rb`'s `default_type`.
- [ ] `newColumnFromField` passes `tableName`, not a pre-resolved string.
- [ ] Any injection the current signature existed to allow is preserved by a
      shape Rails has, or justified at the call site.
- [ ] `pnpm parity:api:calls:args` stays green and the
      `mysql/schema-statements.ts` `new_column_from_field` naming row is gone.
- [ ] MySQL adapter column/schema suites pass on the mysql2 lane.
