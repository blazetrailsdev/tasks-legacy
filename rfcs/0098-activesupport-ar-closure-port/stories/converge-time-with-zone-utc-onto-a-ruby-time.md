---
title: "TimeWithZone#utc answers a ::Time, so localtime is Rails' utc.getlocal"
status: ready
updated: 2026-08-23
rfc: "0098-activesupport-ar-closure-port"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 260
priority: 1
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`ActiveSupport::TimeWithZone#utc` answers a `::Time`
(`activesupport/lib/active_support/time_with_zone.rb:63-65` —
`@utc ||= incorporate_utc_offset(@time, -utc_offset)`), and `getgm`,
`gmtime` and `comparable_time` are its aliases (`:66-68`).

trails' `TimeWithZone#utc` (`packages/activesupport/src/time-with-zone.ts:493`)
answers a `Temporal.Instant` instead. That is what forced the seating hop in
PR #6895: `localtime` is Rails' one-liner `utc.getlocal(utc_offset)`
(`time_with_zone.rb:83-85`), but a `Temporal.Instant` has no `getlocal`, so
trails spells it

```ts
Time.at(new Rational(this.utc().epochNanoseconds, 1_000_000_000n)).getlocal(utcOffset);
```

`@blazetrails/date`'s `Time` now carries `getlocal`
(`packages/date/src/time.ts`), so the only thing standing between that line and
the Rails one is `utc`'s return type.

## Converged shape

`utc()` answers a `::Time` (`@blazetrails/date`'s `Time`), and `getgm()` /
`gmtime()` / `comparableTime()` follow it as the aliases they are. `localtime`
then reads as Rails' `utc.getlocal(utc_offset)` with no seating hop, and
`to_time`'s three arms (`:493-501`) call it directly.

Blast radius to price first: `utc()` is read widely inside `time-with-zone.ts`
(`period`, `toDatetime`, `freeze`, the comparison operators) and by
`core-ext/time-with-zone.test.ts`, which asserts
`toBeInstanceOf(Temporal.Instant)` in several places. Rails asserts
`assert_instance_of Time` on the same values, so those assertions converge with
the reader rather than being worked around.

## Acceptance criteria

- [ ] `TimeWithZone#utc` answers a `::Time`; `getgm` / `gmtime` /
      `comparableTime` are its aliases.
- [ ] `localtime` is `utc.getlocal(utcOffset)` with no epoch reseating.
- [ ] Mirrored assertions in `core-ext/time-with-zone.test.ts` follow Rails'
      `assert_instance_of Time`; no test name touched.
- [ ] `parity:api:calls` / `:args` clean; `parity:test` delta non-negative.
