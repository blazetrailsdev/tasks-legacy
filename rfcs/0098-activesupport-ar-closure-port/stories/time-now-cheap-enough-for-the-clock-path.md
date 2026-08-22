---
title: "Make Time.now cheap enough to sit on the production clock read"
status: ready
updated: 2026-08-22
rfc: "0098-activesupport-ar-closure-port"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 180
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

The unblocker for
[time-helpers-stub-date-and-datetime-clock](../../0098-activesupport-ar-closure-port/stories/time-helpers-stub-date-and-datetime-clock.md),
which cannot converge while `Time.now` is too expensive to sit on trails'
production clock read.

Measured on this host (vitest, 5000 iterations each):

| call                                | ms/op  |
| ----------------------------------- | ------ |
| `Temporal.Now.instant()`            | 0.0075 |
| `Temporal.Now.timeZoneId()`         | 0.117  |
| `instant.toZonedDateTimeISO("UTC")` | 0.0047 |
| `Time.now()`                        | 0.52   |
| `time.toTime()`                     | 0.0017 |

`Time.now` (`packages/date/src/time.ts`) is `#atInstant(Temporal.Now.instant())`,
and `#atInstant` pays `Temporal.Now.timeZoneId()` (0.117ms, an `Intl`
`resolvedOptions()` round trip) plus a full `new Time(...)` — which range-checks
every positional through `obj2ubits`/`validateVtmRange` and builds a
`Temporal.PlainDateTime` — only to have `#instant` and `#utcOffset` overwritten
by the caller immediately after.

That is ~70x a bare `Temporal.Now.instant()`, and
`currentTimeInstant()` (`packages/activesupport/src/time-travel.ts`) is read on
every `TimeWithZone` construction, so `travel_to` stubs an internal `clock`
seat alongside the four receivers Rails stubs rather than routing through
`Time.now` (PR #6872, justified at that call site).

## Converged shape

Two independent wins, neither of which changes an observable:

- Memoize the process time zone id. `Temporal.Now.timeZoneId()` is read on
  every `#atInstant`; MRI reads the process's zone once and caches it
  (`time.c` `find_time_t` / `localtime_with_gmtoff_zone`). Note the memo must be
  per-process and NOT survive a `TZ` change inside a test — `time.trails.test.ts`'s
  `inZone` helper rewrites `TZ` between cases, so expose the reset that
  helper needs.
- Give `#atInstant` a seat that skips the validating constructor. Its inputs
  come from a `Temporal.ZonedDateTime` and are valid by construction, and the
  two fields the constructor computes are overwritten anyway.

## Acceptance criteria

- [ ] `Time.now()` is within a small constant of `Temporal.Now.instant()` —
      target under 0.05ms/op on the numbers above.
- [ ] `packages/date` suite green, including the DST fall-back and `inZone`
      cases in `time.trails.test.ts`.
- [ ] No observable change: `Time.at`, `Time.utc`, `Time.mktime` and
      `Time.now` answer the same zone, offset and sub-second as before.
