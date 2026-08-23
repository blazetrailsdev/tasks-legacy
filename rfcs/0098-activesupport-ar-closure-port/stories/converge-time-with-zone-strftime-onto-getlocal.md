---
title: "TimeWithZone#strftime is getlocal(utc_offset).strftime; its PERMANENT tag's reason is stale"
status: ready
updated: 2026-08-23
rfc: "0098-activesupport-ar-closure-port"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 120
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`ActiveSupport::TimeWithZone#strftime` is
`getlocal(utc_offset).strftime(format)`
(`activesupport/lib/active_support/time_with_zone.rb:225-228`), after the `%Z`
gsub.

trails' `strftime` (`packages/activesupport/src/time-with-zone.ts`) instead
calls `this._local()` and hands the date gem's `strftime` a hand-built field
record. It carries a `@missingRailsCall getlocal — PERMANENT` tag whose stated
reason is:

> trails' `getlocal` answers a `Temporal.PlainDateTime`, which carries no
> `strftime`

**That reason is now false.** `getlocal` answers `@blazetrails/date`'s `Time`
(PR #6895 gave it `getlocal`; PR #6930 removed the seating hop), and that
`Time` has a `strftime` — `packages/date/src/time.ts`, `Time#strftime`. The
PERMANENT tag is a stale ratification standing in front of a reachable
convergence, which is exactly the debt CLAUDE.md's "a documented deviation is
debt, not permission" names.

## Converged shape

```ts
strftime(format: string): string {
  format = format.replace(/((?:^|[^%])(?:%%)*)%Z/g, `$1${this.zone}`);
  return this.getlocal(this.utcOffset).strftime(format);
}
```

Delete the `@missingRailsCall getlocal` tag with it — do not reword it. Check
whether `_local()` still has a caller once this lands; if not, remove it too
(it is a trails-only private with no Rails counterpart).

Watch the `%Z` arm: the `::Time` `getlocal(utcOffset)` answers was built from an
offset, so its own `zone` is `nil` and a `%Z` the gsub left behind prints empty
— which is what Ruby does, and what the current JSDoc already records.

## Acceptance criteria

- [ ] `TimeWithZone#strftime` is Rails' `getlocal(utc_offset).strftime(format)`
      after the `%Z` gsub.
- [ ] The `@missingRailsCall getlocal` tag is deleted, not reworded.
- [ ] `_local()` is removed if it has no remaining caller.
- [ ] `parity:api:calls` clean; existing `strftime` tests unchanged and green.
