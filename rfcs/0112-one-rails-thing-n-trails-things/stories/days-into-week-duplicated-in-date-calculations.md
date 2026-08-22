---
title: "DAYS_INTO_WEEK is declared twice; the Date arm should read the mixin's constant"
status: done
updated: 2026-08-22
rfc: "0112-one-rails-thing-n-trails-things"
cluster: dead-mixin-companions
packages: []
deps: []
deps-rfc: []
est-loc: 40
pr: 6738
claim: "2026-08-19T12:59:52Z"
assignee: "days-into-week-duplicated-in-date-calculations"
blocked-by: null
closed-reason: null
---

## Context

Surfaced while porting `DateAndTime::Calculations`' second half (#6463).

`DAYS_INTO_WEEK` is declared by the mixin
(`activesupport/lib/active_support/core_ext/date_and_time/calculations.rb:8-16`),
and `Date.find_beginning_of_week!` (`core_ext/date/calculations.rb:32-35`)
resolves it through `Date`'s ancestors — `Date` includes the mixin, so there is
exactly one constant.

trails now has two: the mixin's, exported from
`packages/activesupport/src/core-ext/date-and-time/calculations.ts`, and a
private copy in `packages/activesupport/src/core-ext/date/calculations.ts:24-38`
(whose own JSDoc already cites the mixin as its source). #6463 left the copy in
place rather than risk an ESM cycle: `date/calculations.ts` importing the mixin
file, which namespace-imports `date/calculations.js`, closes one.

## Converged shape

`date/calculations.ts` imports `DAYS_INTO_WEEK` from the mixin file and the
private copy is deleted. Verify the cycle both ways with a plain-node import of
the BUILT `dist/**.js` modules as entry modules (a vitest run enters the funnel
module first and masks a TDZ). If the cycle is real, the settled answer is a
zero-import slot module, not a second copy of the constant.

## Acceptance criteria

- [ ] Exactly one `DAYS_INTO_WEEK` declaration in activesupport.
- [ ] `core-ext/date/` and `core-ext/date-and-time/` suites green.
- [ ] Both `dist` entry orders import cleanly under plain node.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
