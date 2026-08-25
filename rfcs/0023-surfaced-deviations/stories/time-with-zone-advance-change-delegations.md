---
title: "time-with-zone-advance-change-delegations"
status: draft
updated: 2026-08-19
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activesupport"
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

# TimeWithZone#advance / #change: port the delegations Rails makes

## Context

Surfaced while burning down `wave-4f-activesupport-residue`. Four call-set
rows in `scripts/api-compare/call-mismatches-exclude/activesupport/time-with-zone.json`
are not name noise — they are two genuine behavioural gaps, now carrying
reviewed reasons that point here.

### 1. `advance` does not delegate (`time_with_zone.rb:430-437`)

```ruby
def advance(options)
  if options.values_at(:years, :weeks, :months, :days).any?
    method_missing(:advance, options)
  else
    utc.advance(options).in_time_zone(time_zone)
  end
end
```

trails' `advance` (`packages/activesupport/src/time-with-zone.ts`) computes
both arms inline off `_local()` and raw epoch-millisecond arithmetic. The
fixed-length arm cannot delegate because trails has no `DateAndTime::Calculations#advance`
core-ext for a `Temporal.Instant` receiver — `packages/activesupport/src/core-ext/date-and-time/calculations.ts`
exports no `advance`. Port that first, then the arm collapses to the Ruby.

### 2. `change` accepts neither `:zone` nor `:offset` (`time_with_zone.rb:390-410`)

Rails' `change` raises `ArgumentError` when both are given, resolves the
survivor through `::Time.find_zone`, and re-seats the result with
`TimeWithZone.new(..., new_zone, ...)`. trails' `ChangeOptions` interface
carries no `zone`/`offset` key at all, so `t.change(zone: "Hawaii")` is a type
error rather than a re-zoned time.

## Acceptance criteria

- [ ] `Time#advance` (the `DateAndTime::Calculations` mixin's `advance`) is
      reachable for trails' `Time` analogue, and `TimeWithZone#advance` is the
      Ruby two-arm body delegating to it.
- [ ] `TimeWithZone#change` accepts `zone` and `offset`, raises
      `ArgumentError, "Can't change both :offset and :zone at the same time: ..."`
      when both are passed, and resolves through `findZone`.
- [ ] The four rows (`advance -> any?`, `advance -> in_time_zone`,
      `change -> find_zone`, `change -> new`) are deleted from the baseline by
      hand and the shard tightened.
- [ ] Rails tests for `advance`/`change` in
      `vendor/rails/activesupport/test/core_ext/time_with_zone_test.rb` are
      ported and green.
