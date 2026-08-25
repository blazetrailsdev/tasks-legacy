---
title: "AdapterSchemaSource hand-projects column flags; Rails passes real Column objects"
status: ready
updated: 2026-07-27
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 150
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`AdapterSchemaSource#columns` (packages/activerecord/src/schema-dumper.ts:238-295)
hand-projects every reflected `Column` into a plain `ColumnInfo` literal, and
duck-types each behavioural flag:

```ts
const isVirtual =
  typeof (col as any).isVirtual === "function"
    ? (col as any).isVirtual()
    : (col as any).virtual === true;
```

Rails has no counterpart. `ActiveRecord::ConnectionAdapters::SchemaDumper`
(vendor/rails/activerecord/lib/active_record/connection_adapters/abstract/schema_dumper.rb)
receives the real `Column` objects from `connection.columns(table)` and calls
`column.virtual?` / `column.virtual_stored?` directly. The projection layer
exists only to also accept plain-object mock sources, which is a trails
invention.

The cost is silent data loss: every flag needs its own hand-written projection
line, and a missing one fails open rather than loud. PR #5177 hit exactly this
— `virtualStored` was never projected, so `sqlite3/schema-dumper.ts:40` read
`column.virtualStored` as `undefined` and dumped _every_ SQLite generated
column as `stored: false`, producing a schema dump that would not round-trip.
Nothing caught it until a Rails-mirrored `test_schema_dumping` was ported.
`isEnum`, `isSerial`, `array`, `unsigned`, `extra` sit on the same footing.

## Acceptance criteria

- Audit the projection for other flags reachable on real `Column` objects but
  dropped on the way into `ColumnInfo` (compare against the fields each dialect
  dumper actually reads).
- Either pass real `Column` objects through to the dialect dumpers like Rails
  does, or make the projection total/type-checked so a newly-read flag cannot
  silently arrive `undefined`.
- Existing dumper suites (schema-dumper, sqlite3/mysql/pg dialect dumpers,
  comment, defaults) stay green.

## Absorbed: `converge-schema-dumper-fk-check-hook-duck-type`

Merged in during the RFC 0023 triage pass (2026-08-18). Original title: "Converge SchemaDumper FK/check-constraint hook duck-type onto Rails' supports\_\* gate"

### Context

`SchemaDumper._hookHost` / `_fkHookHost`
(`packages/activerecord/src/schema-dumper.ts:1067`, `:1080`) duck-type the
dumper's source for a `foreignKeys` / `checkConstraints` function and skip the
section entirely when the lookup fails. Rails has no such probe: it gates on the
adapter capability and then calls the connection method unconditionally —

- `schema_dumper.rb:145` `if @connection.supports_foreign_keys?` … `:317`
  `if (foreign_keys = @connection.foreign_keys(table)).any?`
- `schema_dumper.rb:210` `if @connection.supports_check_constraints?` … `:284`
  `if (check_constraints = @connection.check_constraints(table)).any?`

The duck-type was load-bearing while `SchemaStatements#foreignKeys` returned
`[]` for unsupported adapters. PR #5840 converged that body to
`raise NotImplementedError, "foreign_keys is not implemented"`
(`schema_statements.rb:1103`), and `checkConstraints` already raised, so the
`typeof fn === "function"` test now answers `true` for every adapter (both
methods are mixed into `AbstractAdapter`) and the real "adapter cannot report
these" signal is the raise, not a missing property. The probe is dead weight
that also diverges from the capability gate Rails actually uses.

### Acceptance criteria

- The foreign-key and check-constraint dump sections are gated on
  `supportsForeignKeys()` / `supportsCheckConstraints()` as in Rails, and call
  the connection method directly.
- `_hookHost` and `_fkHookHost` are deleted (or reduced to whatever the
  `AdapterSchemaSource`-vs-source indirection genuinely still needs, with the
  duck-type gone).
- SQLite, MySQL and PostgreSQL lanes green; `schema-dumper.test.ts` cases that
  construct a source with no `foreignKeys` hook are re-derived against the
  capability gate.
