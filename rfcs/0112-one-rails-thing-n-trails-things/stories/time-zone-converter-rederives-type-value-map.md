---
title: "TimeZoneConverter re-derives Type::Value#map with isRangeLike/mapRange instead of calling map"
status: ready
updated: 2026-08-22
rfc: "0112-one-rails-thing-n-trails-things"
cluster: split-stores
packages: []
deps: []
deps-rfc: []
est-loc: 130
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`TimeZoneConverter` has no `Range` or `Array` branch in Rails
(`vendor/rails/activerecord/lib/active_record/attribute_methods/time_zone_conversion.rb:13-50`).
Both `cast` and `convert_time_to_time_zone` end in the same line:

```ruby
map(super) { |v| cast(v) }            # cast, :33
map(value) { |v| convert_time_to_time_zone(v) }   # convert_time_to_time_zone, :48
```

The container polymorphism lives entirely in `Type::Value#map`
(`vendor/rails/activemodel/lib/active_model/type/value.rb:117-119`, identity),
overridden by `OID::Range#map`
(`vendor/rails/activerecord/lib/active_record/connection_adapters/postgresql/oid/range.rb:50-54`)
and `Type::Array#map`. `TimeZoneConverter` never learns what a range is.

trails re-derives that dispatch inside the converter instead
(`packages/activerecord/src/attribute-methods/time-zone-conversion.ts:101,104,118,120,195,197`):
a private `RangeLike` interface, an `isRangeLike()` structural predicate
(`:367`) and a `mapRange()` helper (`:373`) that reconstructs through
`range.constructor`, plus explicit `Array.isArray` arms — six call sites of
machinery Rails does not have.

Both overrides it stands in for are already ported:
`packages/activemodel/src/type/value.ts:94` (`map`) and
`packages/activerecord/src/connection-adapters/postgresql/oid/range.ts:135`
(`RangeType#map`). They are simply not on the path.

Surfaced in PR #6219 (`range-is-an-interface-not-a-class`) while auditing the
structural range predicates left in the tree after `Range` became a class.

## Converged shape

`cast`'s and `deserialize`/`convertTimeToTimeZone`'s container arms collapse to
Rails' single `map` call against the subtype, e.g.

```ts
return this._subtype.map(casted, (v) => this.cast(v));
```

`RangeLike`, `isRangeLike` and `mapRange` are deleted, not rehomed; the
`Array.isArray` arms go with them (`Type::Array#map` covers them once ported —
port it here if it is missing).

Note the bound-specific behaviour the current code carries in the range arm
(`castBoundInZone`, `_subtypeIsUtc`) has to survive as the block body, since
Rails' block is just `{ |v| cast(v) }` and the bound handling in trails exists
because `RangeType.cast` would misparse a bare timestamp string. Check whether
that is itself a divergence in `RangeType.cast` before preserving it.

## Acceptance criteria

- [ ] `time-zone-conversion.ts` has no `isRangeLike`, `mapRange` or `RangeLike`;
      the container arms are `this._subtype.map(value, block)`.
- [ ] `Type::Array#map` is ported if absent, so array columns still round-trip.
- [ ] tsrange/tstzrange and `datetime[]` time-zone tests still pass on PG.
- [ ] `pnpm parity:api:extra --package activerecord` novel count drops; `pnpm parity:api:calls`
      clean (the converter's call set gains `map`).

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
