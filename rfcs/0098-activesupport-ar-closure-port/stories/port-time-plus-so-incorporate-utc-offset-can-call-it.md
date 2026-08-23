---
title: "Port Time#+ so incorporate_utc_offset's else arm is Rails' `time + offset`"
status: in-progress
updated: 2026-08-23
rfc: "0098-activesupport-ar-closure-port"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 130
priority: null
pr: 6958
claim: "2026-08-23T22:26:33Z"
assignee: "api-build-reflows-the-other-tag-familys-jsdoc"
blocked-by: null
closed-reason: null
---

## Context

Ruby's `Time#+` (`ruby/time.c` `time_plus`) is not ported by
`@blazetrails/date`'s `Time` (`packages/date/src/time.ts` — no `plus`/`+`
analogue anywhere in its member list).

`ActiveSupport::TimeWithZone#incorporate_utc_offset`
(`vendor/rails/activesupport/lib/active_support/time_with_zone.rb:562-568`) is
two lines:

```ruby
if time.kind_of?(Date)
  time + Rational(offset, SECONDS_PER_DAY)
else
  time + offset
end
```

PR #6934 moved `@utc` onto a `::Time`, so the `else` arm now has a `::Time`
receiver and no `Time#+` to call. It is spelled through the epoch instead
(`packages/activesupport/src/time-with-zone.ts`, `_incorporateUtcOffset`):

```ts
return Time.at(
  new Rational(time.toTime().toInstant().epochNanoseconds, 1_000_000_000n).add(offset),
).getutc();
```

That is a correct instant but not the Rails body: a reader has to reconstruct
`time + offset` from four calls.

## Converged shape

Port `Time#+` (and its `Time#-` twin, `time.c` `time_minus`, if it falls out of
the same seat) onto `packages/date/src/time.ts`, taking the numeric/`Rational`
seconds MRI takes and answering a `Time` in the receiver's own zone. Then
`_incorporateUtcOffset`'s `else` arm is literally `time.plus(offset)` — one
call, matching time_with_zone.rb:567.

## Acceptance criteria

- [ ] `Time#+` is ported at the Rails/MRI name and semantics (Rational seconds,
      receiver's zone preserved), with `time.c` cited.
- [ ] `TimeWithZone#_incorporateUtcOffset`'s `else` arm calls it directly; the
      `Time.at(...).getutc()` reconstruction is gone.
- [ ] `parity:api:calls` / `:args` clean; activesupport + date suites green.
