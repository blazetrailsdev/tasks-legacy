---
title: "Time#change carries new_sec twice (a float and a rebuilt Rational); Rails carries one Rational"
status: ready
updated: 2026-08-23
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

`Time#change` carries one `new_sec` local, and it is a Rational from the moment
the microseconds fold in
(`activesupport/lib/active_support/core_ext/time/calculations.rb:129-141`):

```ruby
new_sec = options.fetch(:sec, (options[:hour] || options[:min]) ? 0 : sec)
...
new_usec = Rational(new_nsec, 1000)   # or options.fetch(:usec, Rational(nsec, 1000))
raise ArgumentError, "argument out of range" if new_usec >= 1000000
new_sec += Rational(new_usec, 1000000)
```

Every one of the five terminal arms then passes that same `new_sec` (`:145`,
`:147`, `:150`, `:174`, `:176`).

trails' `change` (`packages/activesupport/src/time-ext.ts`) carries it **twice**:

- `newSec`, a JS number, which is floored into `newComponents` for the
  `Temporal` arms;
- `newSecRational`, rebuilt from `newComponents`' second/ms/us/ns, which the
  three `::Time` arms pass.

`newSecRational` is not a Rails name, and two locals stand where Ruby has one.
The values agree at nanosecond granularity today — PR #6930 verified the
observable results against MRI — so this is a naming and decomposition
divergence, not a behavioural one. It is still worth closing: a reviewer
reading the two bodies side by side sees Rails' `new_sec` in five arms and
trails' in none of them.

## Converged shape

One `newSec`, a `Rational`, matching Rails:

- seed it from `options.sec ?? (...)` through `Rational`;
- carry `newUsec` as a `Rational` too — Rails' is
  `Rational(new_nsec, 1000)` / `Rational(nsec, 1000)`, and the
  `>= 1000000` guard compares against it;
- `newSec = newSec.add(newUsec.div(1_000_000))`, the port of
  `new_sec += Rational(new_usec, 1000000)`;
- derive `newComponents`' `second` / `millisecond` / `microsecond` /
  `nanosecond` from that Rational for the `Temporal` arms, rather than the other
  way round.

Note the callers that hand `usec` a float today —
`endOfHour` / `endOfMinute` / `endOfDay` pass `999999999 / 1000`, where Ruby
passes `Rational(999999999, 1000)`. Converging `new_usec` is the moment to look
at whether those should pass a `Rational` too.

Regression guard: the existing `change`/`allDay`/`advance with nsec` tests
already pin the nanosecond edges (`999999998` vs `999999999` truncation was a
real failure during #6930), so they are the baseline this must not move.

## Acceptance criteria

- [ ] `change` carries one `newSec`, a `Rational`, passed by all terminal arms.
- [ ] `newUsec` is a `Rational`, and the `>= 1000000` guard reads against it.
- [ ] `newSecRational` is gone.
- [ ] `core-ext/time-ext.test.ts` and `core-ext/date-ext.test.ts` unchanged and
      green — in particular `advance with nsec` and `all day`.
