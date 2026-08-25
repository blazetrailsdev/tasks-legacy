---
title: "TimeZoneConverter#cast splits the in_time_zone arm and drops the infinite? and rescue arms"
status: draft
updated: 2026-08-22
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 120
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

PR #6848 (story `converge-parse-string-in-zone-onto-user-input-in-time-zone`)
landed the `user_input_in_time_zone` half of this story: the String arm now
calls the subtype's `userInputInTimeZone`, `parseStringInZone` is deleted, and
the `cast` -> `user_input_in_time_zone` call-set baseline row is gone. The
branch-shape half is still open, and is what remains here.

`TimeZoneConverter#cast`
(`vendor/rails/activerecord/lib/active_record/attribute_methods/time_zone_conversion.rb:19-32`)
has exactly four arms:

```ruby
return if value.nil?
if value.is_a?(Hash)                       then set_time_zone_without_conversion(super)
elsif value.respond_to?(:in_time_zone)     then (super(user_input_in_time_zone(value)) || super rescue ArgumentError -> nil)
elsif value.respond_to?(:infinite?) && value.infinite? then value
else                                            map(super) { |v| cast(v) }
end
```

trails (`packages/activerecord/src/attribute-methods/time-zone-conversion.ts`,
`cast`) still splits Rails' single `respond_to?(:in_time_zone)` arm into four
TS-typed arms — `TimeWithZone`, `Temporal.ZonedDateTime`, `Temporal.Instant`,
`Temporal.PlainDateTime` — ahead of the String one, each routing to
`convertTimeToTimeZone` or `setTimeZoneWithoutConversion` instead of through
`userInputInTimeZone`. It is also still missing:

- the `value.respond_to?(:infinite?) && value.infinite?` arm (`:28-29`), so an
  infinite bound falls through to the `map` else-branch; and
- the `rescue ArgumentError` -> `nil` guard wrapping the whole
  `in_time_zone` arm (`:25-27`).

`userInputInTimeZone` is reachable on the subtype chain — `OID::Range` and
`OID::Array` both delegate it (`oid/range.rb:9`, `oid/array.rb:13`), and
`ActiveRecord::Type::Internal::Timezone#user_input_in_time_zone` is ported —
and the String arm already goes through it, so the remaining arms have a
landing place.

## Converged shape

One `respond_to?(:in_time_zone)` arm: a single predicate over the value types
that carry a zone conversion (TimeWithZone / Instant / ZonedDateTime /
PlainDateTime / string), whose body is the one already written for the String
case —

```ts
const casted = this._subtype.cast(this._subtype.userInputInTimeZone(value));
return casted != null && casted !== false ? casted : this._subtype.cast(value);
```

(the explicit falsy check is Ruby's `||`, which falls through on `nil`/`false`
only) — wrapped in the ArgumentError -> null rescue, followed by the
`infinite?` arm and then the existing `map(super) { |v| cast(v) }`
else-branch. The four per-type arms collapse into it.

Note the Hash arm stays first and keeps `setTimeZoneWithoutConversion`; only
the `in_time_zone` arm collapses.

## Acceptance criteria

1. `cast` has Rails' four arms, in Rails' order, including `infinite?` and the
   ArgumentError rescue.
2. The four per-type arms collapse into the single `in_time_zone` arm, which
   reaches `userInputInTimeZone` for every value type — not only strings.
3. `attribute-methods/time-zone-conversion.test.ts`, `multiparameter-attributes`,
   and the PG `range`/`array`/`timestamp` suites stay green on all three
   adapter lanes.
