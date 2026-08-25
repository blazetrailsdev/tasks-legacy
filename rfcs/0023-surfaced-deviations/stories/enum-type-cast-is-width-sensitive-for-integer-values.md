---
title: "EnumType#cast misses a numerically-equal bigint where Ruby's has_value? matches"
status: draft
updated: 2026-08-07
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
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

Ruby has one `Integer`; JS has `number` and `bigint`, and `EnumType#cast`
matches the stored value by identity against its reverse mapping, so a
numerically-equal `bigint` misses:

```ruby
# vendor/rails/activerecord/lib/active_record/enum.rb (EnumType#cast)
def cast(value)
  if mapping.has_key?(value)      then value.to_s
  elsif mapping.has_value?(value) then mapping.key(value)
  else                                 value.presence
  end
end
```

`mapping.has_value?(2)` is true whatever width the DB handed back. trails'
`_reverseMapping` (`packages/activerecord/src/enum.ts:243-250, 282-290`) is a
`Map` keyed by the declared values, so `cast(2n)` falls through to the
`value.presence` arm and returns `"2"` — or `null` and then `String(2n)` via
`castInheritanceColumnValue` (`inheritance.ts:24-38`).

Surfaced by PR #6191. `memberships.type` is `t.integer :type` (schema.rb:787)
driving an _enum_, not STI; widening the sibling `t.references` columns to
bigint turned on SQLite's statement-wide `readBigInts`, the integer `type`
spilled to `2n`, and every eager-loaded membership raised
`SubclassNotFound: '2'`. That PR fixed the _spill_ at the adapter row boundary
(`sqlite3-adapter.ts` `_narrowSpilledBigInts`), which is the right layer for
SQLite — but it leaves `EnumType#cast` itself still width-sensitive, so any
other producer of a wide integer hits the same wall. node-postgres hands `int8`
back as a decimal _string_, so a PG enum column declared `bigint` is the
obvious next one.

## Converged shape

`EnumType#cast`'s value-side lookup matches on numeric equality rather than JS
identity, so `cast(2n)`, `cast(2)` and (where the subtype is integer)
`cast("2")` all resolve to the same label — the single-`Integer` semantics
Ruby's `has_value?` gets for free. Keep the label-side (`mapping.has_key?`) arm
exactly as it is: that one _is_ string-keyed in Ruby too.

Add a regression covering a bigint and a decimal-string value against an
integer-subtype enum; it must fail on the pre-fix baseline.

## Acceptance criteria

- [ ] `EnumType#cast` resolves a numerically-equal `bigint` to its label.
- [ ] The STI/enum path (`castInheritanceColumnValue`) no longer depends on the
      adapter having normalized the width first.
- [ ] Enum + eager-loading suites green on all three lanes.
