---
title: "Arel::Table#name is typed string, rejecting the SqlLiteral/node names Rails accepts"
status: done
updated: 2026-08-25
rfc: "0124-arel-surfaced-deviations"
cluster: null
packages:
  - "arel"
deps: []
deps-rfc: []
est-loc: 90
priority: null
pr: 7054
claim: "2026-08-25T16:56:36Z"
assignee: "arel-star-is-a-shared-const-not-a-per-call-method"
blocked-by: null
closed-reason: null
---

## Context

Surfaced in PR #7029 (`table-and-factory-assertion-parity`) porting
`vendor/rails/activerecord/test/cases/arel/table_test.rb:118-130`:

```ruby
it "should accept literal SQL"  do
  rel = Table.new Arel.sql("generate_series(4, 2)")
  assert_equal Arel.sql("generate_series(4, 2)"), rel.name
end

it "should accept Arel nodes"  do
  node = Arel::Nodes::NamedFunction.new("generate_series", [4, 2])
  rel = Table.new node
  assert_equal node, rel.name
end
```

Rails' `Arel::Table#initialize` (`vendor/rails/activerecord/lib/arel/table.rb`)
stores whatever object it is given as `@name` — a String, a Symbol, an
`Arel.sql` SqlLiteral, or a node — and the two tests above exist precisely to
pin that. trails types both the parameter and the field as `string`
(`packages/arel/src/table.ts:60,64`), so the mirrored tests can only be written
with `as unknown as string` casts at the call site
(`packages/arel/src/table.test.ts`, the two `describe("new")` cases). The
runtime already holds the object correctly; only the type lies.

`Nodes::NamedFunction.new("generate_series", [4, 2])` takes the same treatment:
`expressions` is typed `Node[]` while Rails passes raw Integers.

## Converged shape

`Table#name` and the constructor parameter accept the union Rails accepts, and
the internals that assume a string are audited with it:

- `hash()` (`table.ts:185-192`) walks `this.name.length` / `charCodeAt` —
  mirrors `Arel::Table#hash`, which hashes `@name` whatever it is.
- `alias()` (`table.ts:78-79`) interpolates `${this.name}_2`, matching Ruby's
  `"#{name}_2"`, which works for any object with `to_s`.
- `eql()` (`table.ts:197-202`) compares with `===`; Ruby compares with `==`,
  so a SqlLiteral name needs value equality, not identity.

The three `as unknown as string` / `as unknown as Nodes.Node[]` casts in
`table.test.ts` come out with it.

## Acceptance criteria

- `new Table(Arel.sql(...))` and `new Table(node)` typecheck with no cast, and
  `rel.name` returns the object unchanged.
- The casts in `table.test.ts`'s `describe("new")` cases are deleted.
- `pnpm parity:api` / `parity:test` deltas non-negative; arel extra-surface gate
  unchanged.
