---
title: "extra-surface scores structural interface members against the wrong Ruby file"
status: in-progress
updated: 2026-08-22
rfc: "0117-arel-extra-surface-burndown"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 180
priority: null
pr: 6883
claim: "2026-08-22T21:13:02Z"
assignee: "extra-surface-scores-structural-interface-members"
blocked-by: null
closed-reason: null
---

## Context

After #6878 (`struct-members-not-extracted-as-ruby-methods`),
`pnpm parity:api:extra --package arel` reports `attributes/attribute.ts` as
`2 novel, 2 moved`. The two _moved_ rows left are `tableAlias` and
`typeForAttribute`, and neither is a member of `Arel::Attributes::Attribute`
(`vendor/rails/activerecord/lib/arel/attributes/attribute.rb:5-40`). They are
members of the structural `RelationLike` interface declared in the same TS
file — the duck type `Attribute#relation` is typed against, standing in for
whatever Ruby passes (an `Arel::Table`, whose `type_for_attribute` /
`table_alias` are its own, at `arel/table.rb`).

So they are scored against `attribute.rb` only because they share the file. Two
questions to settle together:

- Should `extra-surface.ts` score members of a structural `interface` at all,
  or only class/function surface? A TS interface has no Ruby counterpart by
  construction — Ruby has no structural types — so every interface member is
  either a false extra row or belongs to the Ruby class the interface stands
  in for.
- If interfaces stay in the population, `tableAlias` / `typeForAttribute`
  should credit against `Arel::Table` (`arel/table.rb`), not `attribute.rb`.

The `[ATTRIBUTE_BRAND]` and `relationName` _novel_ rows in the same file are a
separate, pre-existing question and are out of scope here.

## Acceptance criteria

- A decision, implemented: structural-interface members are either excluded
  from the extra-surface population or attributed to the Ruby entity the
  interface stands in for.
- `pnpm parity:api:extra --package arel` for `attributes/attribute.ts` no
  longer reports `tableAlias` / `typeForAttribute` as moved surface of
  `attribute.rb`.
- No other package's extra-surface totals regress; report the delta.
- `pnpm vitest run scripts/api-compare` green.
