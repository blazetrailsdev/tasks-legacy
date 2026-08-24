---
title: "TimeWithZone#advance advances through Time#advance, not a hand-rolled calendar"
status: in-progress
updated: 2026-08-24
rfc: "0098-activesupport-ar-closure-port"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 200
priority: null
pr: 6966
claim: "2026-08-24T02:09:44Z"
assignee: "api-build-reflows-same-family-tags-split-by-prose"
blocked-by: null
closed-reason: null
---

# `TimeWithZone#advance` should advance through `Time#advance`, not a hand-rolled calendar

## Context

`packages/activesupport/src/time-with-zone.ts` `advance()` (~:820-890) builds
the advanced wall clock by hand: it adds years/months into `year`/`month`,
clamps the day with a local `daysInMonth`, accumulates `weeks * 7 + days` into
`day`, and then calls `this._timeZone.local(...)` with the result.

Rails does none of that. `ActiveSupport::TimeWithZone#advance`
(`vendor/rails/activesupport/lib/active_support/time_with_zone.rb:527-535`)
splits the options: the date parts go to `time.advance(...)` — i.e.
`Time#advance` (`activesupport/lib/active_support/core_ext/time/calculations.rb:190-215`),
which routes the calendar arithmetic through `Date#advance` — and the
`in_time_zone` rebuild; the fixed parts (`hours`/`minutes`/`seconds`) go through
`since`. The day-clamping and month-carry trails does by hand is `Date`'s job in
Rails and is already ported in `@blazetrails/date`.

Surfaced by `move-the-remaining-time-zone-date-callers-onto-ruby-time`
(trails#6959): once `TimeZone#local` builds its wall clock with `Time.utc`
(`values/time_zone.rb:363-366`), which raises `ArgumentError: argument out of
range` on a day outside the month rather than rolling over the way `Date.UTC`
did, `advance` had to carry its accumulated days into the calendar itself. That
carry is a `Temporal.PlainDate.add` at the call site — correct, but it is a
second hand-rolled calendar next to the `daysInMonth` clamp above it, and both
exist only because the body never reaches `Time#advance`.

## Converged shape

`advance` splits its options as Rails does and delegates the date half to
`Time#advance`, dropping the local `daysInMonth` clamp, the year/month carry
loops and the `Temporal.PlainDate` day carry — leaving the fixed-part `since`
arm and the time-zone rebuild.

## Acceptance criteria

- [ ] `advance` reaches `Time#advance` for the date parts and `since` for the
      fixed parts, matching time_with_zone.rb:527-535.
- [ ] The hand-rolled month carry, `daysInMonth` clamp and `PlainDate` day carry
      are gone from `time-with-zone.ts`.
- [ ] `time-with-zone.test.ts` green with no test renamed — in particular
      `advances by days`, `advance days` and `plus Duration.days(5)`, the three
      that fail if the day carry is simply deleted.
- [ ] `parity:api:calls` / `:args` clean.
