---
title: "build_with_expression_from_value drops Rails' nested arg; buildWithValueFromHash adds an invented CTE-name guard"
status: ready
updated: 2026-08-25
rfc: "0113-branch-and-guard-parity"
cluster: invented-arm
packages: []
deps: []
deps-rfc: []
est-loc: 140
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while converging `build_with_value_from_hash` (#6463).

Two divergences in the neighbouring ported bodies
(`packages/activerecord/src/relation/query-methods.ts:2359-2394`):

1. `build_with_expression_from_value`
   (`activerecord/lib/active_record/relation/query_methods.rb:1929-1950`) takes
   a second parameter, `nested = false`, and branches on it: an
   `ActiveRecord::Relation` value yields `value.arel.ast` when nested and
   `value.arel` otherwise; the array arm recurses with `nested = true`. trails'
   `buildWithExpressionFromValue` takes one parameter and always returns the
   AST, so the non-nested arm's `SelectManager` return is lost.

2. `buildWithValueFromHash` throws an invented `ArgumentError`
   (`Invalid CTE name "…": must be a valid SQL identifier …`) for a name that
   does not match `/^[A-Za-z_][A-Za-z0-9_]*$/`. Rails has no such guard — it
   passes the name straight to `TableAlias.new` and lets the visitor quote it.

## Converged shape

- Port the `nested` parameter with Rails' default and both arms, and recurse
  with `nested = true` from the array arm.
- Delete the identifier validation and its error, or — if a real caller depends
  on it — move the check to that caller with the Rails `file:line` that forces
  it.

## Acceptance criteria

- [ ] `buildWithExpressionFromValue` has Rails' arity, parameter name and
      default, and both `Relation` arms.
- [ ] No invented `ArgumentError` remains in `buildWithValueFromHash`.
- [ ] `relation/with.test.ts` green on all three adapters.

## Absorbed: `build-with-value-from-hash-node-and-arg-order`

Merged in during the RFC 0023 triage pass (2026-08-18). Original title: "build-with-value-from-hash-node-and-arg-order"

### Context

Surfaced while burning down RFC 0096 wave-2 naming rows.

`QueryMethods#build_with_value_from_hash`
(`vendor/rails/activerecord/lib/active_record/relation/query_methods.rb:1923-1927`):

```ruby
def build_with_value_from_hash(hash)
  hash.map do |name, value|
    Arel::Nodes::TableAlias.new(build_with_expression_from_value(value), name)
  end
end
```

trails (`packages/activerecord/src/relation/query-methods.ts:2380-2394`)
builds `new Nodes.Cte(name, expr)` — a different Arel node class **and** the
reversed argument order. That makes the RFC 0096 row
(`new`: Ruby `ref:buildWithExpressionFromValue, ref:name` → TS
`ref:name, ref:expr`) an a1 finding, not a rename, so wave 2 leaves it
standing.

The TS body also raises an invented `ArgumentError` on a CTE name that is not
a bare SQL identifier (`query-methods.ts:2386-2390`); Rails has no such check.

### Acceptance criteria

- [ ] `buildWithValueFromHash` constructs the Rails node
      (`Arel::Nodes::TableAlias`) with Rails' argument order
      `(buildWithExpressionFromValue(value), name)`, or the divergence is
      justified at the call site with the Arel-side reason.
- [ ] The invented identifier validation is removed or traced to a Rails
      raise site.
- [ ] `pnpm parity:api:calls:args` stays green and the `query-methods.ts`
      `build_with_value_from_hash` naming row is gone.
- [ ] `with(...)` / CTE tests pass.

## Triage note (2026-08-18)

Merged story; `est-loc` raised to ~140. All three divergences live in the same
two ported bodies at `relation/query-methods.ts:2359-2394`, so they are one
read and one PR.

Sequencing: `with-clause-uses-bespoke-ctes-not-with-values` (0023, ~300 LOC) is
the larger sibling that retires the bespoke `_ctes` array and puts these two
bodies on the live path via `with_values` + `build_with`
(`query_methods.rb:1908-1921`). Land THIS story first — it is the cheaper fix
and leaves that one a pure routing change — and do not fold the two together.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0113-branch-and-guard-parity`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
