---
title: "TimeZone#local/at/now/utc_to_local build UTC instants on a JS Date seat, dropping Rails' Time.utc calls"
status: draft
updated: 2026-08-23
rfc: "0023-surfaced-deviations"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 150
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced by #6891, which ported `DateTime#utc`
(`vendor/rails/activesupport/lib/active_support/core_ext/date_time/calculations.rb:184-191`)
and so put `utc` into the call-set population for the first time. Four
pre-existing omissions in `packages/activesupport/src/values/time-zone.ts`
became visible and had to be baselined
(`scripts/api-compare/call-mismatches-exclude/activesupport/values/time-zone.json`,
rows `at`, `local`, `now`, `utc_to_local`, each `call: "utc"`).

Every one is the same divergence: Rails builds a UTC instant by going through a
`Time` receiver, and trails builds a JS `Date` (or reads one) instead, so the
`Time.utc` / `.utc` call has no receiver to land on.

Rails, `vendor/rails/activesupport/lib/active_support/values/time_zone.rb`:

- `:363-366` `def local(*args)` — `time = Time.utc(*args)`, handed to
  `TimeWithZone.new(nil, self, time)`. trails' `local` (time-zone.ts:919-…)
  builds `new Date(Date.UTC(year, month - 1, ...))`.
- `:379-381` `def at(*args)` — `Time.at(*args).utc.in_time_zone(self)`. trails'
  `at` (time-zone.ts:980-988) reaches the same instant via
  `Time.at(...).getutc()`, ruby/time's own `utc`/`getutc` pair.
- `:516-518` `def now` — `time_now.utc.in_time_zone(self)`. trails' `now`
  (time-zone.ts:896-899) reads `this.timeNow()` as a JS `Date`, already
  absolute, so the `.utc` hop has nothing to convert.
- `:542-547` `def utc_to_local(time)` — the non-offset arm is
  `Time.utc(t.year, t.month, t.day, t.hour, t.min, t.sec, t.sec_fraction * 1_000_000)`.
  trails' `utcToLocal` (time-zone.ts:1055-1060) rebuilds the same components
  through `Date.UTC`.

## Converged shape

Move `TimeZone`'s UTC-instant construction onto `@blazetrails/date`'s `Time`
(`packages/date/src/time.ts`), which already carries an offset, a zone,
`utcOffset`/`isUtc`/`getutc` and — since #6891 — `getlocal`. Each of the four
bodies then makes the call Rails makes: `Time.utc(...)` in `local` and
`utc_to_local`, `.utc` in `at` and `now` (`Time#utc` is `getutc` on an
immutable receiver, which is the language-forced spelling and already how `at`
does it).

## Acceptance criteria

- The four bodies call `Time.utc` / `Time#utc` where Rails does.
- The four rows are DELETED from
  `scripts/api-compare/call-mismatches-exclude/activesupport/values/time-zone.json`
  by hand (the baseline is only-shrink — never `--write`/reseed), and any stale
  high-water mark is narrowed with `pnpm parity:api:calls:tighten`.
- `pnpm parity:api:calls` and `pnpm parity:api:calls:args` green.
- `packages/activesupport/src/core-ext/time-with-zone.test.ts` and
  `values/time-zone` tests stay green; Rails test names unchanged.
