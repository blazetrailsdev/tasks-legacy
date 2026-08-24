---
title: "PostgreSQL and MySQL keep columns overrides Rails lacks, with an inconsistent column_definitions/new_column_from_field pair"
status: draft
updated: 2026-08-24
rfc: "0023-surfaced-deviations"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 220
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

PR #7008 converged the abstract `columns` to Rails' shape —
`column_definitions(table_name).map { |field| new_column_from_field(table_name, field, definitions) }`
(`activerecord/lib/active_record/connection_adapters/abstract/schema_statements.rb:107-113`) —
and deleted the `adapterName` switch that stood in for it. SQLite3 then dropped
its own `columns` (Rails has none) and reached the abstract body.

PostgreSQL and MySQL still carry `columns` overrides Rails does not have, and
their `column_definitions` / `new_column_from_field` pair is not
self-consistent, so the abstract body cannot drive them:

- `packages/activerecord/src/connection-adapters/postgresql/schema-statements-class.ts:588`
  overrides `columns`; its body batch-preloads OIDs and then hand-builds the
  positional field tuple (`:663-678`) before calling
  `newColumnFromField`. `postgresql-adapter.ts:3755` destructures `field` as a
  10-element array, but `schema-statements-class.ts:399` `columnDefinitions`
  returns row hashes. Rails' `column_definitions`
  (`postgresql_adapter.rb:1034-1049`) uses `query(...)`, which returns rows as
  ARRAYS — that is why `new_column_from_field`
  (`postgresql_adapter.rb`, private) destructures positionally.
- `packages/activerecord/src/connection-adapters/abstract-mysql-adapter.ts:184`
  overrides `columns` and calls a free-function `newColumnFromField` via
  `.call(...)` rather than a method on the adapter
  (`mysql/schema_statements.rb` defines it as a private method).

Forcing the abstract body onto either adapter raises
(`TypeError: field is not iterable` on PG, `adapter.newColumnFromField is not a
function` on MySQL) — that is what reddened the PG and MariaDB lanes on #7008
and why `schema-statements-reflection-probes.trails.test.ts` no longer layers
the abstract `columns`/`primaryKey` over the live connection.

## Converged shape

- PG `column_definitions` returns rows in Rails' positional order (or the
  adapter's `new_column_from_field` reads the named row consistently — pick one
  contract and hold it on both sides), `new_column_from_field` becomes a real
  method on the adapter, and the `columns` override is deleted so the abstract
  body drives it. The OID batch preload has no Rails counterpart and needs its
  own justification or removal.
- MySQL: `new_column_from_field` becomes a method on `AbstractMysqlAdapter`
  (`mysql/schema_statements.rb`), and its `columns` override is deleted.
- Re-add `columns`/`primaryKey` to the reflection-probe overlay list in
  `packages/activerecord/src/connection-adapters/abstract/schema-statements-reflection-probes.trails.test.ts`
  once the abstract body runs on all three lanes.

## Acceptance criteria

- [ ] `AbstractMysqlAdapter` and the PostgreSQL schema-statements class define
      no `columns` of their own; the abstract body drives all three adapters.
- [ ] Each adapter's `columnDefinitions` and `newColumnFromField` agree on one
      field contract, cited to the Rails adapter.
- [ ] The reflection-probe test layers `columns` and `primaryKey` again.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.
