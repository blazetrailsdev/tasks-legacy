---
title: "Time#change conflates the elsif zone and trailing utc_offset arms and drops isdst"
status: ready
updated: 2026-08-25
rfc: "0113-branch-and-guard-parity"
cluster: missing-arm
packages: []
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

Rails' `Time#change` has two distinct trailing arms
(`activesupport/lib/active_support/core_ext/time/calculations.rb:173-176`):

```ruby
elsif zone
  ::Time.local(new_sec, new_min, new_hour, new_day, new_month, new_year, nil, nil, isdst, nil)
else
  ::Time.new(new_year, new_month, new_day, new_hour, new_min, new_sec, utc_offset)
end
```

`packages/activesupport/src/time-ext.ts` `change` conflates them: a JS `Date`
receiver unconditionally takes the `local()` helper (the `elsif zone` arm), and
the trailing `else ::Time.new(..., utc_offset)` arm is not ported at all. The
`isdst` argument Rails passes to `Time.local` is also dropped — `local()` builds
through `new Date(y, m-1, ...)`, which resolves an ambiguous DST-fold nominal
time by the host's rule rather than by the receiver's `isdst`.

PR #6246 (the `:offset`/`:nsec`/`utc?` arms) left this untouched; it is
pre-existing, and the `Date` vs `Temporal.ZonedDateTime` receiver split is the
thing that makes the two arms indistinguishable.

## Acceptance criteria

- [ ] The `elsif zone` and trailing `else` arms are separately reachable and
      each matches its Rails line.
- [ ] `local()` threads the receiver's `isdst` the way `::Time.local`'s 9th
      argument does, so a fold-ambiguous nominal time picks the receiver's
      occurrence.
- [ ] Covered by `test_change_preserves_offset_for_local_times_around_end_of_dst`
      (`activesupport/test/core_ext/time_ext_test.rb:472`), currently `it.skip`.

## Absorbed: `time-change-subsecond-rational-ported-as-float`

Merged in during the RFC 0023 triage pass (2026-08-18). Original title: "Time#change ports Rational sub-second arithmetic as lossy float division"

### Context

Rails' `Time#change` keeps the sub-second component as an exact `Rational`
(`activesupport/lib/active_support/core_ext/time/calculations.rb:134`,
`:136`, `:141`):

```ruby
new_usec = Rational(new_nsec, 1000)
new_usec = options.fetch(:usec, ... Rational(nsec, 1000))
new_sec += Rational(new_usec, 1000000)
```

`packages/activesupport/src/time-ext.ts` `change` ports all three as IEEE-754
float division (`newNsec / 1000`, `nsec / 1000`, `newUsec / 1_000_000`), then
recovers nanoseconds with `Math.round((newSec - secFloor) * 1_000_000_000)`.
The round-trip is lossy for sub-second values that are not representable in
binary, so a `usec`/`nsec` that Rails preserves exactly can come back off by a
nanosecond. Callers already lean on the boundary value `999999999 / 1000`
(`endOfDay`, `endOfHour`, `endOfMinute`).

### Acceptance criteria

- [ ] Sub-second arithmetic in `change` is exact for every integer
      `usec`/`nsec` in range — no float round-trip through `newSec`.
- [ ] `endOfDay`/`endOfHour`/`endOfMinute` land on exactly 999999999ns.
- [ ] Rails' locals (`new_usec`, `new_sec`) keep their names and positions.

## Triage note (2026-08-18)

Merged story (~200 LOC est.) — both rows are divergences inside the single
`Time#change` body (`core_ext/time/calculations.rb:130-176`).

Related but deliberately separate: `time-ext-time-arm-sends-utc-and-utc-offset`
(0023, ~250 LOC) covers the call-gate omissions across FIVE `time-ext.ts` Time-arm
methods, of which `change` is one. Whichever lands second should re-read this
body rather than assume it is untouched.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0113-branch-and-guard-parity`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
