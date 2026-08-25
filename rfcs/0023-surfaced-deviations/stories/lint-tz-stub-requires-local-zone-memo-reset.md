---
title: "Lint: a test that rewrites TZ must drop Time's local-zone memo"
status: draft
updated: 2026-08-23
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "date"
deps: []
deps-rfc: []
est-loc: 90
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`Time`'s local-zone memo (`packages/date/src/time.ts:34-48`, `localTimeZoneId` /
`resetLocalTimeZoneId`, the port of MRI's `tzset` cache — `time.c`
`localtime_with_gmtoff_zone`) is invisible to a test that rewrites `TZ`. Rails'
`with_env_tz` (`activesupport/test/time_zone_test_helpers.rb:12-17`) only has to
mutate and restore `ENV["TZ"]` in an `ensure`, because MRI re-reads the zone on
`tzset`; on the trails side every such helper must additionally call
`resetLocalTimeZoneId()` on BOTH sides of the stub, and nothing enforces it.

PR #6944 was the second time this bit: `date-time-ext.test.ts`'s `withEnvTz`
stubbed `TZ` without the reset, so the `localtime` / `getlocal` cases warmed the
memo with `US/Eastern` and every later `Time#change` (`time-ext.ts:446-627`)
read its components in the real zone and rebuilt them through `RubyTime.local`
in the stale one. Five `DateAndTime::Calculations` predicates (`today?`,
`yesterday?` x2, `tomorrow?` x2) went red on main — but only between 20:00 and
24:00 UTC, where the leaked 4-hour offset actually crosses a date boundary, and
never on a host already on Eastern. Three consecutive main reds before it was
found.

Correct callers today: `time-ext.test.ts:66-92`, `string-ext.test.ts:111-120`,
`date-time-ext.test.ts:70-80`, `internal-metadata.trails.test.ts:33-45`.

## Acceptance criteria

- A lint rule flags any `vi.stubEnv("TZ", ...)` / `process.env.TZ = ...` write in
  a `*.test.ts` whose enclosing function or hook does not also call
  `resetLocalTimeZoneId()`, and likewise flags the restore half
  (`vi.unstubAllEnvs()` / the `finally` restore) without it.
- The four call sites above pass the rule unchanged.
- Removing either `resetLocalTimeZoneId()` from `withEnvTz` in
  `date-time-ext.test.ts` reds the rule.

## Verification

`pnpm lint` on a branch with the reset deleted from one helper fails; with all
four helpers intact it passes.
