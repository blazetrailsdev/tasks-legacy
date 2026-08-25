---
title: "abstract quote_default_expression has an undefined arm and a Proc TypeError Rails does not"
status: ready
updated: 2026-08-25
rfc: "0077-quoting-binds-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 120
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

# abstract quote_default_expression has an undefined arm and a Proc arm Rails does not

## Context

Surfaced while landing `sqlite3-quote-default-expression-standalone-skips-serialize`
(PR #7061), which converged the sqlite3 standalone onto `super`. Converging it
put the abstract body it now delegates to in front of me, and that body carries
two arms Rails has not got.

Rails, `vendor/rails/activerecord/lib/active_record/connection_adapters/abstract/quoting.rb:157-164`:

```ruby
def quote_default_expression(value, column) # :nodoc:
  if value.is_a?(Proc)
    value.call
  else
    value = lookup_cast_type(column.sql_type).serialize(value)
    quote(value)
  end
end
```

Two branches, and the Proc branch is a bare `value.call` whose result is
returned as-is.

`packages/activerecord/src/connection-adapters/abstract/quoting.ts:294-320`
(`quoteDefaultExpression`) instead has:

1. `if (value === undefined) return "";` as a THIRD arm, ahead of the Proc
   check. Rails has no such arm — a `nil` default there falls to the `else` and
   is serialized then quoted, reaching `"NULL"`, not `""`. The `""` return also
   silently produces a bare trailing `SET DEFAULT` / an empty DDL fragment at the
   `add_column_options!` call site (`schema_creation.rb:150`) rather than an
   error.
2. A Proc arm that unwraps `isSqlLiteral(result)` and throws
   `TypeError("quoteDefaultExpression expected function default to return a
string or SqlLiteral")` for anything else. Rails returns whatever the Proc
   returned, unexamined. In Ruby `SqlLiteral < String` so the unwrap is a no-op
   there, but the TypeError is a raise site Rails does not have — CLAUDE.md
   ("Errors. Same error class, same message string, same raise site").

The sqlite3 subclass body (`sqlite3/quoting.rb:99-110`) and the PG one carry the
same two invented arms, mirrored from this one — converging the abstract should
sweep those too, or they will keep reintroducing it by copy.

Note the `undefined` arm is load-bearing for at least one caller today:
`abstract-mysql-adapter.ts:755` comments that it normalizes `undefined` to
`null` specifically to dodge "the bare SET that
`quoteDefaultExpression(undefined)` -> `""` would emit". Untangle that before
deleting the arm.

## Converged shape

Two branches, matching rb:157-164 — `Proc` -> `value.call` returned as-is;
everything else -> `lookupCastType(column.sqlType).serialize(value)` then
`quote(value)`. Remove the `undefined` arm and the TypeError; audit the callers
that lean on `""` (start with `abstract-mysql-adapter.ts:755`) and the sqlite3 /
PG subclass bodies that copy the same arms.

## Acceptance criteria

- [ ] `abstract/quoting.ts`'s `quoteDefaultExpression` has exactly Rails' two
      branches; no `undefined` pre-arm, no TypeError raise site.
- [ ] The sqlite3 and PG overrides carry no copy of the removed arms.
- [ ] Callers relying on the `""` return are converged or explicitly repointed,
      `abstract-mysql-adapter.ts:755` included.
- [ ] parity:api / parity:test delta non-negative; all three lanes green.
