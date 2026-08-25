---
title: "Enum subtype is inferred by name and looked up in a registry, not taken from the reflected column type"
status: ready
updated: 2026-07-27
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 150
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Rails never resolves an enum's storage subtype by name. `EnumType.new(name,
mapping, subtype)` receives the attribute's already-reflected `Type::Value`,
threaded in by `decorate_attributes([name]) { |_name, subtype| ... }`
(`vendor/rails/activerecord/lib/active_record/enum.rb:240-249`) — the decorator
argument IS the column's real type, so a `decimal` column's enum delegates to the
reflected `Type::Decimal` with its actual precision/scale.

trails instead infers a _name_ from the mapping's value shapes
(`inferSubtype`, `packages/activerecord/src/enum.ts:30-43` — returns
`"integer"` / `"boolean"` / `"string"`) and looks that name up in a registry
(`subtypeInstance`, enum.ts:46-58), falling back to `new IntegerType()` when the
lookup throws. PR #5196 switched that lookup from ActiveModel's `typeRegistry`
to ActiveRecord's `arTypeLookup` so it follows the same registry as
`attribute()`, but did not change the shape: it is still name-inference plus a
registry lookup, not the reflected column type.

Consequences: a column whose real type carries options (decimal
precision/scale, string limit, an adapter-specific or user-registered type)
gets a bare default-constructed type instead, and the `catch` silently masks a
failed lookup as `IntegerType`. Noted during #5196 review as pre-existing and
explicitly out of that PR's scope.

## Acceptance criteria

- The enum subtype comes from the reflected attribute type the way
  `enum.rb:240-249` threads it through `decorate_attributes`, not from
  `inferSubtype` + a registry lookup by name.
- `inferSubtype` / `subtypeInstance` are deleted, or reduced to a documented
  fallback that only runs where Rails genuinely has no reflected type, with the
  silent `catch -> IntegerType` removed or justified at the call site.
- A test proves a decimal-backed enum keeps the column's precision/scale, and a
  string-backed enum keeps its limit — currently both are lost.
- `enum.test.ts` and `enum.trails.test.ts` stay green.
