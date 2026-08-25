---
title: "TimeWithZone#to_time/#localtime/#strftime: the getlocal seam"
status: draft
updated: 2026-08-19
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activesupport"
  - "date"
deps: []
deps-rfc: []
est-loc: 260
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

# TimeWithZone#to_time / #localtime / #strftime: the `getlocal` seam

## Context

Surfaced burning down `wave-4f-activesupport-residue` (PR #6731). Three
`kind: "set"` rows in
`scripts/api-compare/call-mismatches-exclude/activesupport/time-with-zone.json`
share one root cause and now carry per-site reasons pointing here: trails'
`utc()` answers a `Temporal.Instant` and `getlocal()` answers a
`Temporal.PlainDateTime`, neither of which is the `::Time` Rails' `getlocal`
returns and calls back into.

### 1. `to_time` drops the `preserve_timezone` three-arm split

`vendor/rails/activesupport/lib/active_support/time_with_zone.rb:493-501`:

```ruby
def to_time
  if preserve_timezone == :zone
    @to_time_with_timezone ||= getlocal(time_zone)
  elsif preserve_timezone
    @to_time_with_instance_offset ||= getlocal(utc_offset)
  else
    @to_time_with_system_offset ||= getlocal
  end
end
```

trails' `toTime()` (`packages/activesupport/src/time-with-zone.ts`) is
`return this.utc()` — one arm, no memo, no `preserve_timezone` read. The
deprecation warning for the Rails 8.1 default flip is already emitted elsewhere
in the suite, so the config seam exists; the three arms do not.

### 2. `localtime` / `strftime` cannot route through `getlocal`

- `localtime` is `utc.getlocal(utc_offset)` (`time_with_zone.rb:83-85`);
  trails builds the shifted wall clock with
  `toZonedDateTimeISO("UTC").toPlainDateTime()` because `Temporal.Instant` has
  no `getlocal`.
- `strftime` is `getlocal(utc_offset).strftime(format)`
  (`time_with_zone.rb:225-228`); trails calls `@blazetrails/date`'s `strftime`
  directly with the field hash it would have read off that `::Time`, because
  `Temporal.PlainDateTime` carries no `strftime`.

Converging both means giving the `Time` analogue a `getlocal` that answers
something with `strftime` — i.e. leaning on `packages/date/src/time.ts` rather
than raw Temporal on this seam. Do that first, then the two bodies collapse to
the Ruby.

## Acceptance criteria

- [ ] `TimeWithZone#toTime` implements all three `preserve_timezone` arms with
      the Rails memo fields, and the Rails tests for it in
      `vendor/rails/activesupport/test/core_ext/time_with_zone_test.rb` are
      ported and green.
- [ ] `localtime` and `strftime` route through a `getlocal` whose result
      carries `strftime`, matching the Ruby bodies line for line.
- [ ] The three rows (`to_time -> getlocal`, `localtime -> getlocal`,
      `strftime -> getlocal`) are deleted from the baseline by hand and the
      shard tightened with `pnpm parity:api:calls:tighten`.
