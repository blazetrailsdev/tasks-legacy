---
title: "change_column quotes its default against a synthetic { sqlType } object, not a column"
status: done
updated: 2026-08-25
rfc: "0077-quoting-binds-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 200
priority: null
pr: 7056
claim: "2026-08-25T17:17:28Z"
assignee: "converge-token-for-class-attribute-stores"
blocked-by: null
closed-reason: null
---

## Context

Rails' `change_column` never quotes a default against an ad-hoc column. Each
adapter builds a real `ChangeColumnDefinition` and lets `add_column_options!`
pass `options[:column]`:

```ruby
sql << " DEFAULT #{quote_default_expression(options[:default], options[:column])}" if options_include_default?(options)
```

(`vendor/rails/activerecord/lib/active_record/connection_adapters/abstract/schema_creation.rb:151`;
PG resolves the live column at `postgresql/schema_creation.rb:102`.)

trails' `packages/activerecord/src/connection-adapters/abstract/schema-statements.ts`
`changeColumn` is instead one method that branches on `this.adapterName`
("mysql2" / "postgres" / else) and, since PR #7035 made `column` required,
passes a synthetic object literal `{ sqlType }` built from the new type string
rather than a column. That object satisfies the `{ sqlType?: string | null }`
duck type `quoteDefaultExpression` reads, so `lookup_cast_type(column.sql_type)`
resolves correctly — but it is not a Column, carries none of a Column's other
fields, and any future use of `column.type` / `column.array?` in the quoter (PG
already reads both, `postgresql/quoting.rb:159-161`) silently sees `undefined`.

The `adapterName` switch itself is the deeper deviation: Rails has no such
branch, it has per-adapter `change_column` overrides.

Surfaced during PR #7035 (`quote-default-expression-column-is-required`).

## Converged shape

Split `changeColumn`'s adapter branches into per-adapter overrides as Rails has
them, and route each through its `SchemaCreation` visitor so the default is
quoted against the real `ChangeColumnDefinition` column
(`abstract/schema_creation.rb:151`), retiring the `{ sqlType }` literal.

## Acceptance criteria

- [ ] No `this.adapterName ===` branching in `changeColumn`; the MySQL and PG
      arms live in their adapters as Rails has them.
- [ ] The default is quoted against a real column/ColumnDefinition, not an
      object literal; the `{ sqlType }` stand-in is gone.
- [ ] PG array and uuid column defaults still quote correctly through
      `postgresql/quoting.rb:159-161`.
- [ ] Migration and schema suites pass on all three lanes; parity deltas
      non-negative.
