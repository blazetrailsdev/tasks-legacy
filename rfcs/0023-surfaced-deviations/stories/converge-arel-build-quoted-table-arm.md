---
title: "build_quoted drops Rails' Arel::Table pass-through arm"
status: draft
updated: 2026-08-24
rfc: "0023-surfaced-deviations"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 70
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`Arel::Nodes.build_quoted` is missing Rails' `Arel::Table` pass-through arm.

Rails (`activerecord/lib/arel/nodes/casted.rb:47-51`):

```ruby
def self.build_quoted(other, attribute = nil)
  case other
  when Arel::Nodes::Node, Arel::Attributes::Attribute, Arel::Table, Arel::SelectManager, Arel::Nodes::SqlLiteral, ActiveModel::Attribute
    other
  else
    ...Casted.new(other, attribute) / Quoted.new(other)
  end
end
```

trails (`packages/arel/src/nodes/casted.ts:27-40`) handles `Node`,
`Arel::Attributes::Attribute`, `ActiveModel::Attribute` and — since #7003 — the
`SelectManager` arm (matched on its `ast` duck-type, returning the manager
itself). **`Arel::Table` has no arm**: a Table is not a `Node` and has no `ast`,
so it falls through and is wrapped in `Quoted`/`Casted`. Rails returns it
unchanged for `visit_Arel_Table` to render.

This is the same class of bug #7003 fixed for `SelectManager`, where the missing
arm cost subqueries their parens (`"users"."karma" > SELECT AVG(...)` instead of
`> (SELECT AVG(...))`). The Table arm is the remaining half.

The two existing sibling stories quote this same `case` statement but are about
different arms and do NOT cover this one:
`arel-build-quoted-passes-model-attribute-unwrapped`,
`arel-attribute-quoted-node-nil-builds-casted`.

## Converged shape

Return `other` unchanged when it is an `Arel::Table`. Note the constraint that
shaped the SelectManager arm: `casted.ts` is a node module and importing
`Table` from it closes a require cycle, so match structurally (as the
SelectManager arm matches on `.ast`) or route through the existing
`node-slots.ts` slot mechanism the `_Attribute` arm already uses —
`_setBuildQuoted` / `_Attribute` are the established shape for exactly this.

## Acceptance criteria

- `buildQuoted(table)` returns the Table itself, not a `Quoted`/`Casted` wrap.
- A test covers a Table reaching a predicate that routes through `quotedNode`
  (e.g. `attr.eq(table)`), asserting the rendered SQL rather than only the node
  class.
- No require cycle introduced: verify with a plain-node import of the BUILT
  `dist/**.js` modules as entry modules, not a vitest run (a vitest run enters
  the funnel module first and masks a TDZ).
- `pnpm vitest run packages/arel` green; `pnpm parity:api:calls` and
  `parity:api:calls:args` clean.
