---
title: "Move values/time-zone.ts' remaining Date-seated period callers onto ::Time, deleting utcTimeFrom"
status: claimed
updated: 2026-08-23
rfc: "0098-activesupport-ar-closure-port"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 200
priority: null
pr: null
claim: "2026-08-23T22:40:29Z"
assignee: "collapse-the-activerecord-secure-password-duplicate"
blocked-by: null
closed-reason: null
---

## Context

PR #6934 moved TZInfo's three period readers onto the `::Time` their Ruby
counterparts take — `period_for_utc`, `periods_for_local`, `period_for_local`
(`TZInfo::Timezone`, reached from
`vendor/rails/activesupport/lib/active_support/values/time_zone.rb:555-565`).

Three callers inside `packages/activesupport/src/values/time-zone.ts` still hold
a `Date` and convert at the boundary through a module-private `utcTimeFrom`:

- `Timezone#periodsForLocal` — `periods.push(this.periodForUtc(utcTimeFrom(utc)))`,
  where `utc` is a `new Date(localMs - offset * 1000)`;
- `Timezone#localToUtc` — `this.periodForLocal(utcTimeFrom(time), dst)`
  (`local_to_utc`, time_zone.rb:551-552);
- `TimeZone#local` — `this.periodForLocal(utcTimeFrom(time))`, over the
  `new Date(Date.UTC(...))` wall clock it builds (time_zone.rb:363-366, which
  Rails builds with `Time.utc(*args)` — the standing
  `@missingRailsCall utc` at that call site names the same gap).

`utcTimeFrom` carries a `@noRailsEquivalent CONVERGEABLE` tag pointing here.

## Converged shape

The `TimeZone` / `Timezone` seat itself is a `::Time`, not a `Date`:
`TimeZone#local` builds its wall clock with `Time.utc(*args)`,
`Timezone#localToUtc` takes and answers `::Time`, and `periodsForLocal`'s
candidate instants are `Time` values. `utcTimeFrom` then has no callers and is
deleted, and `TimeZone#local`'s `@missingRailsCall utc` retires with it.

Note `localToUtc` also answers a `Date` today; Ruby's `local_to_utc` answers a
`::Time`, so its return type moves with the argument.

## Acceptance criteria

- [ ] `Timezone#localToUtc` takes and answers the `::Time` `local_to_utc` does.
- [ ] `TimeZone#local` builds its wall clock through `Time.utc`, retiring the
      `@missingRailsCall utc` at that call site.
- [ ] `utcTimeFrom` is deleted from `values/time-zone.ts`.
- [ ] `parity:api:calls` / `:args` clean; activesupport suite green on all lanes.
