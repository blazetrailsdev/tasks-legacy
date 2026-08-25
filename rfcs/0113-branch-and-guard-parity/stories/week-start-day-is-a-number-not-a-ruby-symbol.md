---
title: "Week-start day is spelled as a number (and elsewhere a string), not Rails' day-name Symbol"
status: ready
updated: 2026-08-25
rfc: "0113-branch-and-guard-parity"
cluster: missing-arm
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

Surfaced while converging `core_ext/date_ext_test.rb`'s assertions (#6644),
where `test_all_week`'s second assertion is `Date.new(2011,6,7).all_week(:sunday)`
and had nothing to pass it.

Rails spells a week-start day as a **day-name Symbol** everywhere in
`DateAndTime::Calculations`, resolved through the `DAYS_INTO_WEEK` map:

- `DAYS_INTO_WEEK = { sunday: 0, monday: 1, ... }`
  (`activesupport/lib/active_support/core_ext/date_and_time/calculations.rb:8-16`)
- `days_to_week_start(start_day = Date.beginning_of_week)` →
  `DAYS_INTO_WEEK.fetch(start_day)` (`:258-260`)
- `beginning_of_week(start_day = Date.beginning_of_week)` (`:267`)
- `end_of_week(start_day = Date.beginning_of_week)` (`:283`)
- `all_week(start_day = Date.beginning_of_week)` (`:316-317`)
- `next_week(given_day_in_next_week = Date.beginning_of_week, ...)` (`:200`)
- `prev_week(start_day = Date.beginning_of_week, ...)` (`:223`)

`packages/activesupport/src/time-ext.ts` spells the same parameter three
different ways, none of them Rails':

- `beginningOfWeek(date, startDay = 1)` / `endOfWeek(date, startDay = 1)` /
  `allWeek(date, startDay = 1)` — a **number**, so a caller writes `0` for
  Rails' `:sunday`.
- `lastWeek(date, startDay = "monday")` / `prevWeek` — a **string** day name.

So the port is not only diverged from Rails, it is inconsistent with itself: the
same concept is a number in one function and a string in its neighbour, and no
arm reads the configured `Date.beginning_of_week` default that every Rails
signature names.

PR #6644 added `startDay` to `allWeek` to match the number form its two callees
already used — deliberately matching the local convention rather than widening
the deviation, and noted as such in review. This story is the convergence.

## Converged shape

Every week-start parameter in `time-ext.ts` (and the `date-and-time/calculations.ts`
mixin arm, if it carries one) takes the Ruby Symbol spelling — a lowercase
day-name string, `"sunday"`..`"saturday"`, per CLAUDE.md's "a Ruby Symbol is a JS
string" rule — resolved through the single `DAYS_INTO_WEEK` map rather than a
bare number. The default is the configured `beginningOfWeek()`
(`core-ext/date/calculations.ts`), not a hardcoded `1`. Callers passing a number
are updated; the numeric arm is deleted, not kept alongside.

Note `days-into-week-duplicated-in-date-calculations` (RFC 0023) is adjacent —
it converges the _duplicated_ `DAYS_INTO_WEEK` constant. Landing that first, or
together, avoids resolving Symbols through two different copies of the map.

## Acceptance criteria

- No week-start parameter in `activesupport` is spelled as a number.
- `beginningOfWeek` / `endOfWeek` / `allWeek` / `lastWeek` / `prevWeek` /
  `nextWeek` all take the same day-name spelling and the same default.
- `date-ext.test.ts`'s `all week` passes `"sunday"` where Rails passes `:sunday`.
- `pnpm parity:api` arity/option deltas non-negative; touched suites green.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0113-branch-and-guard-parity`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
