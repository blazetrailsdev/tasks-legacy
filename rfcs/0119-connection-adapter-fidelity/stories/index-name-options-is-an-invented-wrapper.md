---
title: "Delete the trails-only indexNameOptions wrapper; index_name takes column_names"
status: draft
updated: 2026-08-11
rfc: "0119-connection-adapter-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 60
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while converging `schema-statements.ts` for RFC 0096 in PR #6356.

`schema_statements.rb:1479-1482` writes

```ruby
column_names = index_column_names(column_name)
index_name = name&.to_s
index_name ||= index_name(table_name, column_names)
```

— `index_name` takes the COLUMN NAMES directly.

`packages/activerecord/src/connection-adapters/abstract/schema-statements.ts`
(`addIndexOptions`) instead writes
`this.indexName(tableName, this.indexNameOptions(columnNames))`, routing through
a trails-only `indexNameOptions(columnNames)` helper (`:2192`) that wraps the
names in `{ column: … }`. That is both an invented call-site conversion and an
invented helper Rails does not have — the kind of extra surface
`pnpm parity:api:extra` measures.

Check the other `indexName` callers while you are here: several pass a
`{ column: … }` object (`renameTableIndexes`, `renameColumnIndexes`), so the
divergence may be in `indexName`'s own signature rather than at this one site.

## Converged shape

`indexName(tableName, columnNames)` taking what `schema_statements.rb:1482`
passes, with `indexNameOptions` deleted.

## Acceptance criteria

1. `indexName` takes Rails' argument, verified against
   `schema_statements.rb:1482` and `index_name`'s own definition.
2. `indexNameOptions` is deleted (not tagged `@noRailsEquivalent`), and every
   caller updated.
3. `pnpm parity:api:extra` drops by the removed name; `pnpm parity:api:calls:args`
   green.
