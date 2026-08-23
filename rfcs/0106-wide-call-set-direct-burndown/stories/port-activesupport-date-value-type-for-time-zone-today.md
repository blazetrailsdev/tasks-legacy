---
title: "port-activesupport-date-value-type-for-time-zone-today"
status: claimed
updated: 2026-08-23
rfc: "0106-wide-call-set-direct-burndown"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: null
claim: "2026-08-23T00:27:26Z"
assignee: "port-activesupport-date-value-type-for-time-zone-today"
blocked-by: null
closed-reason: null
---

## Context

`TimeZone#today` (`vendor/rails/activesupport/lib/active_support/values/time_zone.rb:521-523`)
is `tzinfo.now.to_date` — it answers a `::Date`. trails has no `Date` value type
on this seam, so `today()` in
`packages/activesupport/src/values/time-zone.ts` returns the
`{ year, month, day }` triple its callers (`tomorrow`, `yesterday`) consume,
read off `now()`, and never calls `to_date`.

Surfaced while migrating RFC 0106 wave 5f call-set baseline rows to
`@missingRailsCall` receipts: the row's reviewed reason is not a language
shortcoming, so its tag carries `CONVERGEABLE (story
port-activesupport-date-value-type-for-time-zone-today)` rather than
`PERMANENT`. The tag is the receipt for THIS story; closing it retires the tag.

## Acceptance criteria

- [ ] `TimeZone#today` (and `tomorrow` / `yesterday`) answer a ported `Date`
      value type rather than an ad-hoc `{ year, month, day }` object, with
      `to_date` actually called at the `time_zone.rb:521-523` site.
- [ ] The `@missingRailsCall to_date` tag on `today()` in
      `packages/activesupport/src/values/time-zone.ts` is deleted, not reworded.
- [ ] `pnpm parity:api:calls` and `pnpm parity:api:calls:args` green.
