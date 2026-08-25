---
title: "Attribute's type-caster delegations cast name back to string; Rails passes it untyped"
status: ready
updated: 2026-08-25
rfc: "0124-arel-surfaced-deviations"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 40
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Shipped in PR #7054 (RFC 0124, `arel-table-star-stores-a-string-not-a-sqlliteral`).

`Attribute#name` now carries what Rails' `Struct.new :relation, :name`
(`activerecord/lib/arel/attributes/attribute.rb:5`) carries — `string | Node`,
so `table[Arel.star]` seats a `SqlLiteral`. Two delegations in the same file
still narrow it back with a cast:

```ts
// packages/arel/src/attributes/attribute.ts
get typeCaster(): unknown {
  return this.relation.typeForAttribute(this.name as string);
}

typeCastForDatabase(value: unknown): unknown {
  return this.relation.typeCastForDatabase(this.name as string, value);
}
```

Rails passes `name` through untyped — `attribute.rb:12-18` is
`relation.type_cast_for_database(name, value)` and
`relation.type_for_attribute(name)`, and `Arel::Table`'s own delegations
(`table.rb:100-107`) hand it straight to the duck-typed caster. The cast is
correct at runtime (a star attribute is never type-cast) but it is a type-level
deviation that the next reader has to re-derive.

## Converged shape

Widen the two `RelationLike` members and `Arel::Table`'s matching methods to
Ruby's untyped name, so the delegation is a bare pass-through with no cast:

- `RelationLike.typeForAttribute(name: string | Node)`
- `RelationLike.typeCastForDatabase(attrName: string | Node, value: unknown)`
- `Table#typeForAttribute` / `Table#typeCastForDatabase` (`table.rb:100-107`)

The `TypeCaster` interface (`table.ts`, already `@noRailsEquivalent PERMANENT`)
is where the string boundary belongs, if one is still needed — Rails' caster is
duck-typed, so a single documented narrowing there beats two casts at the call
sites. Check the activerecord implementers (`type-caster/`) for fallout.

## Acceptance criteria

- No `as string` cast on `this.name` in `packages/arel/src/attributes/attribute.ts`.
- `pnpm parity:api --package arel` / `parity:api:calls:args` deltas non-negative;
  arel extra-surface gate unchanged.
- `pnpm vitest run packages/arel` and the activerecord type-caster tests green.
