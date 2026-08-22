---
title: "Time.new/Time.now should take MRI's in: and precision: kwargs"
status: ready
updated: 2026-08-22
rfc: "0098-activesupport-ar-closure-port"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 160
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

PR #6872 ported `Time.new` (`time.c` `time_s_init`) so `travel_to` has the
receiver Rails stubs (`time_helpers.rb:180-187`). It carries only the six
positionals:

`packages/date/src/time.ts` — `static new(year?, month, day, hour, min, sec)`.

MRI's `Time.new` also takes two keywords, and `Time.now` takes one:

- `in:` — the zone the time is built in. `Time.new(2020, 1, 1, in: "+05:00")`
  is `2020-01-01 00:00:00 +0500`. The positional 7th argument (the `zone`
  parameter trails' `Time` constructor already has) is the older spelling of
  the same thing, so the keyword is a second entry to a seat that exists.
- `precision:` — the sub-second digits kept. `Time.new(precision: 3)` with no
  other argument is the current time truncated to the millisecond, which is why
  Rails' own test uses it as a value that must NOT equal the travelled time.

The gap is load-bearing in a Rails test we ported. `time_travel_test.rb:70` and
`:86` both assert:

```ruby
assert_not_equal expected_time, Time.new(precision: 3)
```

Those two assertion arms are dropped in
`packages/activesupport/src/time-travel.test.ts` (`time helper travel to`,
`time helper travel to with block`), noted in the merge commit as "not
carried". They cannot be restored until `precision:` exists.

## Converged shape

`Time.new(year = nil, ..., in: nil, precision: nil)` and
`Time.now(in: nil, precision: nil)`, kwargs spelled the trails way (a trailing
options object), with `in` routed to the constructor's existing `zone`
parameter and `precision` truncating the `Rational` sub-second before
`#atInstant` seats it.

Note that `travel_to`'s `Time.new` STUB must keep forwarding whatever it is
handed to the original method (`time_helpers.rb:184-185`
`Time.send(stub.original_method, *args, **options)`) — trails forwards `...args`
through `apply`, so an options object rides along once the signature takes one.

## Acceptance criteria

- [ ] `Time.new` and `Time.now` take `in:` and `precision:`, matching MRI on
      `Time.new(2020,1,1, in: "+05:00").utc_offset == 18000` and
      `Time.new(precision: 3).nsec % 1_000_000 == 0`.
- [ ] The two `assert_not_equal expected_time, Time.new(precision: 3)` arms are
      restored in `time-travel.test.ts` (`time_travel_test.rb:70,86`).
- [ ] `parity:test:assertions` green.
