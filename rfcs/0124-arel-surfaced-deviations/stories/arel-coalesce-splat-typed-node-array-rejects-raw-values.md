---
title: "FactoryMethods#coalesce types its splat Node[], rejecting the raw values Rails passes through"
status: claimed
updated: 2026-08-25
rfc: "0124-arel-surfaced-deviations"
cluster: null
packages:
  - "arel"
deps: []
deps-rfc: []
est-loc: 60
priority: null
pr: null
claim: "2026-08-25T15:39:01Z"
assignee: "arel-case-reader-readonly-vs-attr-accessor"
blocked-by: null
closed-reason: null
---

## Context

Surfaced in PR #7029 porting
`vendor/rails/activerecord/test/cases/arel/factory_methods_test.rb:69-76`:

```ruby
def test_coalesce
  relation = Table.new(:users)
  field_node = relation[:active]
  coalesce = @factory.coalesce field_node, 0
  assert_instance_of Nodes::NamedFunction, coalesce
  assert_equal "COALESCE", coalesce.name
  assert_equal [field_node, 0], coalesce.expressions
end
```

Rails' `FactoryMethods#coalesce`
(`vendor/rails/activerecord/lib/arel/factory_methods.rb:45-47`) is
`NamedFunction.new("COALESCE", exprs)` — it wraps **nothing**, and the test's
`assert_equal [field_node, 0]` is the assertion that pins that pass-through: a
raw Integer reaches `expressions` unmodified.

trails types the splat as `Node[]`
(`packages/arel/src/factory-methods.ts:46`, and `NamedFunction#expressions`
likewise), so the mirrored test can only pass Rails' `0` through a
`0 as unknown as Nodes.Node` cast. The runtime is already correct — the value
is stored untouched — but the signature says a caller may not do what every
Rails caller does.

Sibling instance in the same file: `createJoin` / `createAnd` already admit
`string` because Rails' own tests pass bare Symbols
(`factory_methods.rb:64-65` comment in trails), so the widening precedent for
this module exists; `coalesce` was missed.

## Converged shape

`coalesce(...exprs)` (and `NamedFunction`'s `expressions`) admit the values
Rails admits rather than `Node[]` only — Rails' NamedFunction stores whatever
the caller passed and the visitor quotes at compile time. The cast in
`factory-methods.test.ts`'s `coalesce` case comes out with it, along with the
call-site note explaining it.

## Acceptance criteria

- `factory.coalesce(fieldNode, 0)` typechecks with no cast and
  `coalesce.expressions` deep-equals `[fieldNode, 0]`.
- The cast and its call-site comment in `factory-methods.test.ts` are deleted.
- arel extra-surface gate and `parity:api:calls:args` unchanged or tighter.
