---
title: "Cte/TableAlias name and relation shadow Binary's left/right instead of aliasing them"
status: ready
updated: 2026-08-25
rfc: "0124-arel-surfaced-deviations"
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

Surfaced in #7043 while converging arel's `readonly` readers onto Rails'
`attr_accessor` (RFC 0124, `arel-case-reader-readonly-vs-attr-accessor`).

In Rails, `Cte` and `TableAlias` do not have `name`/`relation` fields at all —
they are **aliases of `Binary`'s slots**:

```ruby
# vendor/rails/activerecord/lib/arel/nodes/cte.rb:6-7
class Cte < Arel::Nodes::Binary
  alias :name :left
  alias :relation :right

# vendor/rails/activerecord/lib/arel/nodes/table_alias.rb:6-8
class TableAlias < Arel::Nodes::Binary
  alias :name :right
  alias :relation :left
  alias :table_alias :name
```

`Binary` declares `attr_accessor :left, :right` (`binary.rb:6`), so in Ruby a
write through either spelling is a write to the one ivar.

trails declares them as **own fields that shadow** the Binary slots and
assigns both in the constructor (`packages/arel/src/nodes/cte.ts:16-25`,
`packages/arel/src/nodes/table-alias.ts:25-33`):

```ts
super(name, relation);
this.name = name;
this.relation = relation;
```

That was inert while all four were `readonly`. #7043 made them writable, per
`binary.rb:6`, which is correct — and it makes the divergence observable:
`cte.left = x` leaves `cte.name` stale, and `tableAlias.name = "u2"` leaves
`tableAlias.right` stale. Rails has one slot per pair; trails has two that can
disagree. `Binary#eql`/`hash` read `left`/`right` (`binary.rb:19-27`), so a
write through the aliased spelling is also invisible to node equality.

Note `TableAlias#tableAlias` is already a getter over `this.name`
(table-alias.ts:36-38) — the alias shape this story wants, applied to one of
the three.

## Converged shape

Replace the four own fields with accessor pairs over `left`/`right`, in the
Rails alias direction (`Cte`: name→left, relation→right; `TableAlias`:
name→right, relation→left), and drop the redundant constructor assignments —
`super(...)` already seats both. Keep each reader's current declared type
(`string | SqlLiteral`, `Node | SelectManager`) on the getter so no call site
has to widen.

## Acceptance criteria

- [ ] `Cte#name`/`#relation` and `TableAlias#name`/`#relation` are accessors
      over `left`/`right`, matching `cte.rb:6-7` and `table_alias.rb:6-8`.
- [ ] A test pins that writing either spelling is visible through the other,
      and that `eql` sees the write (fails on the pre-change baseline).
- [ ] `pnpm parity:api` / `parity:api:extra --package arel` deltas
      non-negative; arel suite green.
