---
title: "travel_to's input arms should build a Time, and use change/getlocal"
status: ready
updated: 2026-08-22
rfc: "0098-activesupport-ar-closure-port"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 220
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

PR #6872 converged `travel_to`'s stub half onto Rails' four receivers. Its
INPUT half is still trails-shaped.

Rails (`vendor/rails/activesupport/lib/active_support/testing/time_helpers.rb:162-175`):

```ruby
if date_or_time.is_a?(Date) && !date_or_time.is_a?(DateTime)
  now = date_or_time.midnight.to_time
elsif date_or_time.is_a?(String)
  now = Time.zone.parse(date_or_time)
else
  now = date_or_time
  now = now.to_time unless now.is_a?(Time)
end

now = now.change(usec: 0) unless with_usec
now = now.getlocal
```

Every arm lands on a `Time`, and the two normalizations are `Time#change` and
`Time#getlocal`. trails
(`packages/activesupport/src/testing/time-helpers.ts:174-197`) lands each arm on
a `Temporal.Instant` in a local named `instant`, floors the sub-second with
`Temporal.Instant.fromEpochMilliseconds(Math.floor(ms / 1000) * 1000)` instead
of `change(usec: 0)`, and spells `getlocal` as
`Time.at(new Rational(instant.epochNanoseconds, 1_000_000_000n))`.

Consequences visible today:

- The Rails local `now` is split into `instant` + `now`, so the body reads with
  two names where Rails has one.
- `change` and `getlocal` are absent from the call set (neither is flagged
  today, but neither is made either).
- The extra `at` call site shifts the `at` call-argument pairing: the
  `ref:now` row on `at` is reported as a `naming` mismatch
  (`parity:api:calls:args:report`) purely because trails' FIRST `at` is the
  `getlocal` substitute rather than the `Time.now` stub body's `at(now)`.

## Converged shape

Each arm builds a `Time` directly, so the method carries Rails' single `now`:

- Ruby-`Date` arm: `midnight(dateOrTime)` then `.toTime()` seated as a `Time`.
- String arm: `Time.zone.parse` (already ported) seated as a `Time`.
- else arm: `now.to_time unless now.is_a?(Time)` — the JS-`Date` and
  `Temporal.Instant` inputs are the `to_time` half.

Then port `Time#change` (`activesupport/lib/active_support/core_ext/time/calculations.rb:83-114`)
and `Time#getlocal` (`time.c` `time_localtime_m` — same instant, local zone)
onto `@blazetrails/date`'s `Time` if they are not there yet, and spell the two
normalizations as `now.change({ usec: 0 })` and `now.getlocal()`.

## Acceptance criteria

- [ ] `travelTo` carries Rails' single `now` local, built as a `Time` by every
      input arm.
- [ ] `change(usec: 0)` and `getlocal` are called where Rails calls them, not
      substituted with epoch arithmetic.
- [ ] `parity:api:calls` / `:calls:args` green, and the `at` `naming` row in
      `parity:api:calls:args:report` for `travel_to` is gone.
- [ ] `time-travel.test.ts` and `packages/date` suites stay green.
