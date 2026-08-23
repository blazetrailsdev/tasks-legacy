---
title: "time-zone-local-builds-its-wall-clock-through-time"
status: draft
updated: 2026-08-23
rfc: "0023-surfaced-deviations"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`time_zone.rb:363-366` `TimeZone#local` is `Time.utc(*args)` followed by
`from_utc_to_local`/`TimeWithZone.new`. trails' `local`
(`packages/activesupport/src/values/time-zone.ts`) builds the wall clock as
`new Date(Date.UTC(...))` instead, because trails has no `Time` seat in this
package, so the `Time.utc` call has no receiver.

Converging is the TimeZone-seat port — `local` should build its wall clock
through the ported `Time` (`packages/date` `time.ts`) the way `at` already
reaches `Time#getutc`. The call now sits at the call site as a
`@missingRailsCall` receipt opened
`CONVERGEABLE (story time-zone-local-builds-its-wall-clock-through-time)`,
migrated out of `call-mismatches-exclude/activesupport/values/time-zone.json`
by RFC 0106 wave 5g.

## Acceptance criteria

- [ ] `TimeZone#local` builds its wall clock through the ported `Time`, matching
      `time_zone.rb:363-366`.
- [ ] The `@missingRailsCall utc` receipt on `local` is deleted, not reworded.
- [ ] `pnpm parity:api:calls` and `pnpm parity:api:calls:args` green.
