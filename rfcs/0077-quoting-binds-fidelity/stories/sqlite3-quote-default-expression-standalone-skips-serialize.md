---
title: "sqlite3 quote_default_expression standalone skips super's cast-type serialize"
status: ready
updated: 2026-08-25
rfc: "0077-quoting-binds-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 80
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Rails' SQLite3 `quote_default_expression` handles only the Proc arm itself and
delegates everything else to the abstract body:

```ruby
def quote_default_expression(value, column) # :nodoc:
  if value.is_a?(Proc)
    value = value.call
    value.match?(/\A\w+\(.*\)\z/) ? "(#{value})" : value
  else
    super
  end
end
```

(`vendor/rails/activerecord/lib/active_record/connection_adapters/sqlite3/quoting.rb:99-110`.)
That `super` is `abstract/quoting.rb:157-164`, whose non-Proc arm is
`value = lookup_cast_type(column.sql_type).serialize(value)` then `quote(value)`.

trails' standalone module function
`packages/activerecord/src/connection-adapters/sqlite3/quoting.ts`
`quoteDefaultExpression` re-implements the whole method instead of delegating,
and its non-Proc arm is a bare `quote.call(this, value)` — the `serialize`
step at rb:161 is missing, and the `column` parameter is unused (spelled
`_column` since PR #7035 made it required).

The adapter-level override in `sqlite3-adapter.ts` DOES call
`super.quoteDefaultExpression(value, column)`, so the live adapter path
serializes correctly; it is the standalone that diverges. Any caller reaching
the module function directly (and its tests) gets the unserialized value.

Surfaced during PR #7035 (`quote-default-expression-column-is-required`) while
making `column` required across the surface.

## Converged shape

The standalone keeps only Rails' Proc arm and delegates the `else` to the
abstract `quoteDefaultExpression` (the trails spelling of `super`), passing
`column` through, so `lookup_cast_type(column.sql_type).serialize` runs. The
duplicated `value === undefined` / `value === null` pre-arms should be checked
against Rails too — `abstract/quoting.rb:157-164` has neither.

## Acceptance criteria

- [ ] `sqlite3/quoting.ts`'s `quoteDefaultExpression` non-Proc arm delegates to
      the abstract body rather than calling `quote` directly (sqlite3/quoting.rb:107).
- [ ] `column` is used, not `_column`; the cast-type serialize at
      `abstract/quoting.rb:161` runs for SQLite defaults.
- [ ] A test covers a default whose serialized form differs from its raw form
      (e.g. a `json` column default) through the standalone.
- [ ] parity:api / parity:test delta non-negative; all three lanes green.
