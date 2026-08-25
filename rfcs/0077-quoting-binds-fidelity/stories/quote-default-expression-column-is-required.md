---
title: "quote_default_expression's column is required in Rails, optional in trails"
status: claimed
updated: 2026-08-25
rfc: "0077-quoting-binds-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 90
priority: 2
pr: null
claim: "2026-08-25T14:10:32Z"
assignee: "attribute-dup-must-redup-mutable-value"
blocked-by: null
closed-reason: null
---

## Context

Rails' `quote_default_expression(value, column)`
(`activerecord/lib/active_record/connection_adapters/abstract/quoting.rb:157-164`)
takes a **required** `column` and dispatches
`lookup_cast_type(column.sql_type).serialize(value)` unconditionally.

PR #6291 (`adapterless-schema-quoters-force-lookup-cast-type-guards`) removed the
`this.lookupCastType?.(…)` optional-call guard and the raw-sqlType fallback, so
the dispatch is now unconditional as in Rails. What survives is the **signature**:
`column?` is still optional in
`packages/activerecord/src/connection-adapters/abstract/quoting.ts`
`quoteDefaultExpression`, guarded by `if (column != null)`, because trails' DDL
callers reach `SET DEFAULT` with no column in hand — e.g.
`packages/activerecord/src/connection-adapters/abstract/schema-statements.ts:600`,
`:613`, `:622`, `:689` and `abstract/schema-creation.ts:319`, all of which call
`quoteDefaultExpression(value)` with one argument.

Rails' equivalents always have the column: `add_column_options!`
(`abstract/schema_creation.rb:150`) passes `options[:column]`, and
`change_column_default` resolves the column before quoting.

The same optionality is mirrored on `AbstractAdapter#quoteDefaultExpression` and
the PG/SQLite overrides, so a value quoted through a column-less caller silently
skips `serialize`.

## Converged shape

Make `column` required by giving those DDL call sites the column they are
changing (they know the table and column name; `schema-statements.ts`'s
`change_column_default` path already resolves a column object elsewhere). Then
drop the `if (column != null)` guard so the body is Rails' three lines.

## Acceptance criteria

- [ ] `quoteDefaultExpression(value, column)` — `column` required on the abstract
      standalone, `AbstractAdapter`, and the PG/SQLite overrides.
- [ ] The `if (column != null)` guard is deleted; `lookup_cast_type(...).serialize`
      runs for every non-Proc, non-SqlLiteral default as at rb:161.
- [ ] Schema/DDL default quoting unchanged across all three lanes, including
      binary and array defaults; `sql-default.test.ts`, `column-definition.test.ts`
      and the migration suites pass.
