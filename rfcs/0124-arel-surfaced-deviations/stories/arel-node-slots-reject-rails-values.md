---
title: "Arel node slots typed narrower than the Ruby slot (Table#get, Function expressions, Cte body)"
status: done
updated: 2026-08-25
rfc: "0124-arel-surfaced-deviations"
cluster: null
packages:
  - "arel"
deps: []
deps-rfc: []
est-loc: 120
priority: null
pr: 7043
claim: "2026-08-25T15:39:01Z"
assignee: "arel-case-reader-readonly-vs-attr-accessor"
blocked-by: null
closed-reason: null
---

## Context

Three arel node slots are typed narrower than the Ruby slot accepts, so a
faithful port of a Rails test cannot be written without an `as unknown as` cast.
All three were hit while converging `visitors/to_sql_test.rb` in #7022, and each
cast is in that file today:

1. **`Table#get` refuses a node name.** Ruby's `Table#[]` is
   `def [](name, table = self)` (`arel/lib/arel/table.rb:110-113`) and
   `to_sql_test.rb:50` calls `@table[Arel.star]` — a `Nodes::Star`, not a
   String. trails types it `get(name: string | null, ...)`
   (`packages/arel/src/table.ts:176`), so the port reads
   `table.get(star as unknown as string)`.
2. **`Function` expressions refuse raw values.** Ruby
   `Nodes::NamedFunction.new("generate_series", [4, 2])`
   (`to_sql_test.rb:865`) seats bare Integers, which the visitor renders through
   `visit_Integer`. trails types `expressions: Node[]`
   (`packages/arel/src/nodes/named-function.ts:15`), so the port reads
   `[4, 2] as unknown as Nodes.Node[]`.
3. **`Cte` body refuses a manager.** Ruby
   `Nodes::Cte.new("foo", Table.new(:bar).project(Arel.star))`
   (`to_sql_test.rb:1008`) seats a `SelectManager`; trails requires the `.ast`.

The repo already has the right vocabulary for this: `NodeOrValue` is used for
exactly these slots elsewhere (`packages/arel/src/math.ts:33` and the
`Math`/`Predications` operands), so this is an inconsistency in which slots got
it, not a missing idea.

## Converged shape

Widen the three slots to what the Ruby slot admits — `NodeOrValue` (or
`Node | string | null` for `Table#get`, matching `table.rb:110`) — and delete
the three casts in `packages/arel/src/visitors/to-sql.test.ts`. The casts are
the acceptance signal: grep the file for `as unknown as` and it should be empty
for these three.

## Acceptance criteria

- `Table#get` accepts a node name, per `arel/lib/arel/table.rb:110-113`.
- `Function`/`NamedFunction` expressions accept raw values, per
  `arel/lib/arel/nodes/function.rb` + the `visit_Integer` path.
- `Cte` accepts a `SelectManager` body, per `arel/lib/arel/nodes/cte.rb`.
- The three `as unknown as` casts in `to-sql.test.ts` are gone; arel and
  activerecord suites green; `parity:api` / `parity:api:extra` deltas
  non-negative.
