---
title: "PG OID::Range#map builds a bespoke Range class where Rails builds ::Range"
status: ready
updated: 2026-08-25
rfc: "0119-connection-adapter-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 200
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`OID::Range#map`
(`vendor/rails/activerecord/lib/active_record/connection_adapters/postgresql/oid/range.rb:50-54`)
builds a **core Ruby Range**:

```ruby
def map(value) # :nodoc:
  new_begin = yield(value.begin)
  new_end = yield(value.end)
  ::Range.new(new_begin, new_end, value.exclude_end?)
end
```

and `force_equality?` (`range.rb:56-58`) is `value.is_a?(::Range)`. The whole
PG range type deals in `::Range`; the only class `range.rb` declares is the
_type_ (`OID::Range < Type::Value`).

trails declares a second, bespoke value class in the same file —
`packages/activerecord/src/connection-adapters/postgresql/oid/range.ts:23`
`export class Range { begin, end, excludeEnd }` — with `RangeType` (`:68`) as
the type beside it. `map` (`:135`) and `isForceEquality` (`:140`) then build
and test _that_ class rather than the `::Range` analogue.

Since PR #6219 (`range-is-an-interface-not-a-class`) the `::Range` analogue
exists: `Range` in `packages/activesupport/src/range-ext.ts` is a real class
with `begin`, `end`, `excludeEnd`, and the `core_ext/range/*` reopenings. Two
parallel range classes is what forces every consumer to duck-type — see
`time-zone-conversion.ts`'s `isRangeLike`, which exists precisely because it
must accept either.

## Converged shape

`oid/range.ts` declares only the type. `map` returns
`new Range(...)` from `@blazetrails/activesupport` and `isForceEquality` is
`value instanceof Range` against the same class, matching `range.rb:50-58`.
The bespoke `export class Range` is deleted; its consumers (PG range casting,
quoting, the range handler, `time-zone-conversion.ts`) move to the
activesupport class.

Check `Range`'s export from `packages/activerecord`'s index while doing this —
a rename here changes a public name.

## Acceptance criteria

- [ ] `connection-adapters/postgresql/oid/range.ts` declares no `Range` value
      class; `RangeType#map` builds `@blazetrails/activesupport`'s `Range` and
      `isForceEquality` is `instanceof` against it.
- [ ] PG range column tests (`tsrange`, `int4range`, `daterange`, …) pass
      unchanged.
- [ ] `pnpm parity:api:extra --package activerecord` novel count drops.
- [ ] `pnpm parity:api` / `pnpm parity:test` deltas non-negative.
