---
title: "Move utc_to_local onto the ::Time it takes and answers, retiring its @missingRailsCall utc"
status: done
updated: 2026-08-24
rfc: "0098-activesupport-ar-closure-port"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 180
priority: null
pr: 6966
claim: "2026-08-24T02:09:44Z"
assignee: "api-build-reflows-same-family-tags-split-by-prose"
blocked-by: null
closed-reason: null
---

# Move `utc_to_local` onto the `::Time` it takes and answers

## Context

`move-the-remaining-time-zone-date-callers-onto-ruby-time` (trails#6959) moved
`Timezone#localToUtc`, `Timezone#periodsForLocal` and `TimeZone#local` onto
`::Time` and deleted the module-private `utcTimeFrom` that had been converting
at the boundary. Its sibling reader was out of that story's scope and is the
last `Date`-seated member of the pair:

    Timezone#utcToLocal  (packages/activesupport/src/values/time-zone.ts:~730)
      utcToLocal(time: Date | Temporal.Instant): Temporal.ZonedDateTime

    TimeZone#utcToLocal  (same file, the delegating half)

Ruby's `TZInfo::Timezone#utc_to_local` takes a `::Time` and, as of tzinfo 2,
answers the local time carrying a non-zero UTC offset. Rails reaches it from
`ActiveSupport::TimeZone#utc_to_local`
(`vendor/rails/activesupport/lib/active_support/values/time_zone.rb:540-548`),
whose `utc_to_local_returns_utc_offset_times` arm then either hands that value
back as is or rebuilds it with
`Time.utc(t.year, t.month, t.day, t.hour, t.min, t.sec, t.sec_fraction * 1_000_000)`
(`:545`) — a `Time.utc` call that has no receiver in trails today because the
value on hand is a `Date`. The standing `@missingRailsCall utc — PERMANENT` at
`values/time-zone.ts:~1082` names exactly that gap, and it is not permanent: it
is the same convergence #6959 performed for `TimeZone#local`.

`toDate()` (`values/time-zone.ts:430`) is the remaining `Time`→`Date` funnel
this seat leans on.

## Converged shape

`Timezone#utcToLocal` takes the `::Time` `utc_to_local` takes, and
`TimeZone#utcToLocal`'s non-offset arm builds its result through `Time.utc`
(time_zone.rb:545) rather than `new Date(Date.UTC(...))` — retiring the
`@missingRailsCall utc` at that call site, as
`move-the-remaining-time-zone-date-callers-onto-ruby-time` retired the one on
`TimeZone#local`. Callers that still hold a `Date` convert at their own seat,
not inside this file.

## Acceptance criteria

- [ ] `Timezone#utcToLocal` takes the `::Time` `utc_to_local` does.
- [ ] `TimeZone#utcToLocal` builds its rebuilt arm through `Time.utc`, and the
      `@missingRailsCall utc` on it is deleted.
- [ ] `parity:api:calls` / `:args` clean with no new baseline rows.
- [ ] `time-zone.test.ts`, `time-zone.trails.test.ts` and
      `time-with-zone.test.ts` green, test names unchanged.
