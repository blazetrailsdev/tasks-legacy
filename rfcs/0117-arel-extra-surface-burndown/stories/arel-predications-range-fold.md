---
title: "Fold predications-range.ts back into predications.ts and delete it"
status: done
updated: 2026-08-22
rfc: "0117-arel-extra-surface-burndown"
cluster: null
packages: ["arel"]
deps: []
deps-rfc: []
est-loc: 220
priority: 4
pr: 6856
claim: "2026-08-22T12:20:33Z"
assignee: "arel-operator-spellings-in-conventions"
blocked-by: null
closed-reason: null
---

## Context

`packages/arel/src/predications-range.ts` (228 lines) is the **single largest
contributor** to arel's extra surface: 41 extras — 8 novel, 33 moved — of the
package's 258, and it has **no Rails counterpart file at all**
(`rubyFile: null` in `pnpm parity:api:extra --package arel --json`).

Its own header says what it is: an extraction of Rails' `Predications`
private helpers plus the `between` / `not_between` decision tree, pulled out
so `Predications` (the mixin) and `Attribute` (the class-side overrides) can
share one copy.

Rails has no such file. The helpers are **private methods on
`Arel::Predications` itself**:
`vendor/rails/activerecord/lib/arel/predications.rb` — `between`,
`not_between`, and the private `infinity?`, `unboundable?`, `open_ended?`.

Novel names (`predications-range.ts`):
`parseRange:88`, `infinitySign:121`, `unboundableSign:147`, `isNilBound:176`,
`betweenFromRange:182`, `notBetweenFromRange:208`, plus `begin`, `excludeEnd`.

Moved names (33): `accept`, `and`, `cast`, `coalesce`, `createAnd`,
`createFalse`, `createJoin`, `createOn`, `createStringJoin`,
`createTableAlias`, `createTrue`, `end`, `eq`, `eql`, `fetchAttribute`,
`grouping`, `gt`, `gteq`, `hash`, `in`, `invert`, `isEquality`, `isInfinity`,
`isOpenEnded`, `isUnboundable`, `lower`, `lt`, `lteq`, `not`, `notIn`, `or`,
`quotedNode`, `toSql` — these are the structural-type members of the
`RangeHost` / `RangePredicates` interfaces the file declares, which exist only
because the file exists.

## Approach

Fold the whole file back into `packages/arel/src/predications.ts` and delete
it. Triage category 3 ("if Rails inlines it, inline it") plus category 1.

- `infinitySign` / `unboundableSign` / `isNilBound` collapse into
  `isInfinity` / `isUnboundable` / `isOpenEnded` at their Rails signatures.
  Note the Ruby predicates return _values_, not booleans (CLAUDE.md,
  "Predicates") — check each call site before narrowing a return type.
- `betweenFromRange` / `notBetweenFromRange` inline into `between` /
  `notBetween` in `predications.ts`.
- `parseRange` is the Ruby `Range` protocol; Ruby gets `begin` / `end` /
  `exclude_end?` from the language. If a shared range shim genuinely survives,
  it belongs in the file that owns the Ruby concept, not a new arel file, and
  it is a candidate for the RFC's tag budget — but try inlining first.
- `Attribute#between` / `#notBetween`
  (`packages/arel/src/attributes/attribute.ts`) call the folded methods rather
  than a shared helper module.
- The `RangeHost` / `RangePredicates` interfaces die with the file; that is
  what retires the 33 moved rows.

## Acceptance criteria

- `packages/arel/src/predications-range.ts` deleted.
- `pnpm parity:api:extra --package arel` total drops by **41** (258 → 217 in
  isolation), novel by 8; `noCounterpartFiles` drops 12 → 11.
- `pnpm parity:api` arel deltas non-negative; `predications.rb`'s matched
  method count does not fall.
- `pnpm vitest run packages/arel` green, in particular the `between` /
  `notBetween` range cases.
- At most **one** new `@noRailsEquivalent` tag, and only if a range shim
  genuinely survives inlining.
