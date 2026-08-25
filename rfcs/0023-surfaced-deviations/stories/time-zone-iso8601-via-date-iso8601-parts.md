---
title: "TimeZone#iso8601: parse through Date._iso8601 / Date.ordinal"
status: draft
updated: 2026-08-19
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activesupport"
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

# TimeZone#iso8601: parse through Date.\_iso8601 / Date.ordinal

## Context

Surfaced burning down `wave-4f-activesupport-residue` (PR #6731). The
`iso8601 -> ordinal` row in
`scripts/api-compare/call-mismatches-exclude/activesupport/values/time-zone.json`
now carries a per-site reason pointing here.

`vendor/rails/activesupport/lib/active_support/values/time_zone.rb:396-425`
parses by _decomposing_ the string, not by matching it:

```ruby
parts = Date._iso8601(str)
year = parts.fetch(:year)
if parts.key?(:yday)
  ordinal_date = Date.ordinal(year, parts.fetch(:yday))
  month = ordinal_date.month
  day   = ordinal_date.day
else
  month = parts.fetch(:mon)
  day   = parts.fetch(:mday)
end
time = Time.new(year, month, day, parts.fetch(:hour, 0), parts.fetch(:min, 0),
                parts.fetch(:sec, 0) + parts.fetch(:sec_fraction, 0),
                parts.fetch(:offset, 0))
parts[:offset] ? TimeWithZone.new(time.utc, self) : TimeWithZone.new(nil, self, time)
```

trails' `iso8601` (`packages/activesupport/src/values/time-zone.ts`) instead
recognises the ordinal `YYDDD` form with its own regex, converts it with
day-of-year arithmetic, validates the general form with a second regex, and
then hands the string to `parse`. Two consequences beyond the missing call:

- The `KeyError` `parts.fetch(:year)` / `fetch(:mon)` raises on a partial
  string is not reproduced — trails raises a generic `"invalid date"` from a
  regex miss instead.
- The `parts[:offset]` branch that decides between `TimeWithZone.new(time.utc, self)`
  and `TimeWithZone.new(nil, self, time)` is collapsed into `parse`.

Note `Date._iso8601` is the same primitive
`0023-surfaced-deviations/duration-iso8601-parser-extraction` wants; check
whether that story lands it first.

## Acceptance criteria

- [ ] `Date._iso8601` and `Date.ordinal` exist in `@blazetrails/date` (or the
      former is reused from the Duration parser story), and `TimeZone#iso8601`
      is the Ruby body line for line, including the `fetch` defaults and the
      `parts[:offset]` two-arm construction.
- [ ] `KeyError` propagates from the `fetch` calls where Ruby raises it, rather
      than a blanket `"invalid date"`.
- [ ] The `iso8601 -> ordinal` row is deleted from the baseline by hand and the
      shard tightened with `pnpm parity:api:calls:tighten`.
