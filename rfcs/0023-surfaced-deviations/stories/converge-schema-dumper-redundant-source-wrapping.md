---
title: "Drop the redundant AdapterSchemaSource wrapping in SchemaDumper.dump / dumpTableSchema"
status: draft
updated: 2026-08-25
rfc: "0023-surfaced-deviations"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 25
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Left over from #7014, which moved the adapter→`SchemaSource` bridging into
`SchemaDumper`'s constructor (`packages/activerecord/src/schema-dumper.ts:456`)
so `create_schema_dumper(options)` could pass `self` the way Rails does
(`activerecord/lib/active_record/connection_adapters/abstract/schema_statements.rb:1541`).

Two callers still build the bridge themselves, and now do it redundantly:

- `schema-dumper.ts:563` — `dump` constructs `new AdapterSchemaSource(pool)`
  before calling `this.create(source, options)`, which would bridge the adapter
  itself. Rails' `dump` just yields the connection
  (`activerecord/lib/active_record/schema_dumper.rb:43-48`).
- `schema-dumper.ts:596` — `dumpTableSchema` computes `wrappedSource` for the
  same reason.

Neither is wrong, both are dead weight, and the duplicated `isDatabaseAdapter`
branch is the kind of second bridging site the constructor change exists to
remove.

## Acceptance criteria

1. `dump` and `dumpTableSchema` hand the connection/adapter straight to
   `create`, with no local `AdapterSchemaSource` construction.
2. `dump`'s body reads as `schema_dumper.rb:43-48` does.
3. SQLite, PostgreSQL and MySQL/MariaDB lanes green.
