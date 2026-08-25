---
title: "time-ext.ts's Time arm sends utc?/utc_offset instead of reading getTimezoneOffset() inline"
status: draft
updated: 2026-08-17
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activesupport"
  - "date"
deps: []
deps-rfc: []
est-loc: 250
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

PR #6628 took `utc?` and `utc_offset` off the `date_time/calculations.rb`
scoped skip group (they are now ported on the DateTime receiver). That put both
names back into the call-gate population, which surfaced five PRE-EXISTING
omissions in `packages/activesupport/src/time-ext.ts` — its `Time` arm never
calls them:

- `Time#change` (`core_ext/time/calculations.rb:130-176`) — Rails reads
  `utc?` and `utc_offset` to decide which of `::Time.utc` / `::Time.local` /
  `::Time.new(..., utc_offset)` rebuilds the receiver.
- `Time#formatted_offset` (`core_ext/time/conversions.rb:69-71`) —
  `utc? && alternate_utc_string || TimeZone.seconds_to_utc_offset(utc_offset, colon)`.
  The port reads `-date.getTimezoneOffset() * 60` inline for both.
- `Time#to_time` (`core_ext/time/compatibility.rb:13-15`) — reaches
  `getlocal(utc_offset)`.

They are baselined in
`scripts/api-compare/call-mismatches-exclude/activesupport/time-ext.json` with
the reason "a JS `Date` is an instant that carries no offset of its own". That
is a burndown row, not a settled decision: `@blazetrails/date`'s `Time`
(`packages/date/src/time.ts`) DOES carry `utcOffset` and `zone`, and the
DateTime arm now proves the receiver-with-an-offset shape works.

## Converged shape

Give the `Time` arm the same treatment `date_time/conversions.rb` just got:
either widen its receiver to the ruby/date `Time` that carries an offset, or
port `Time#utc?` / `Time#utc_offset` over the `Date` seat and call them from
`change` / `formatted_offset` / `to_time` instead of reading
`getTimezoneOffset()` inline. Delete the five baseline rows as they converge —
the baseline is only-shrink, so retire them one by one rather than reseeding.

Rails anchors: `core_ext/time/calculations.rb:130-176`,
`core_ext/time/conversions.rb:69-71`, `core_ext/time/compatibility.rb:13-15`.

## Acceptance criteria

- [ ] `change` / `formatted_offset` / `to_time` in `time-ext.ts` send `utc?`
      and `utc_offset` where Rails does.
- [ ] All five `utc?` / `utc_offset` rows are gone from
      `call-mismatches-exclude/activesupport/time-ext.json`, with the stale
      high-water mark tightened via `parity:api:calls:tighten`, never a reseed.
- [ ] `pnpm parity:api` / `pnpm parity:test` deltas non-negative.
