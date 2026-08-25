---
title: "Port core_ext/time/calculations.rb onto the Time class, not free functions"
status: ready
updated: 2026-08-25
rfc: "0113-branch-and-guard-parity"
cluster: missing-arm
deps: []
deps-rfc: []
est-loc: 400
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced by #6160 (`api-compare-orphan-buckets-activesupport-calculations`),
which pointed `activesupport:core_ext/time/calculations.rb` at
`packages/activesupport/src/time-ext.ts` and made its 57 methods visible for the
first time. **21 match, 30 still read missing.**

The deviation is one of shape, not of coverage. Rails reopens the `Time` class:

- `vendor/rails/activesupport/lib/active_support/core_ext/time/calculations.rb:11`
  — `class Time`, with class methods `===` (:18), `days_in_month` (:24),
  `days_in_year` (:34), `current` (:39), `at_with_coercion` (:45),
  `rfc3339` (:69), and instance methods `seconds_since_midnight` (:91),
  `seconds_until_end_of_day` (:100), `sec_fraction` (:107), `change` (:123),
  `advance` (:194), `ago` (:220), `since` (:225), `beginning_of_day` (:238),
  `middle_of_day` (:246), `end_of_day` (:256), `beginning_of_hour` (:267),
  `end_of_hour` (:273), `beginning_of_minute` (:283), `end_of_minute` (:289),
  the `*_with_duration` / `*_with_coercion` operator pairs (:297-356), and
  `prev_day`/`next_day`/`prev_month`/`next_month`/`prev_year`/`next_year`
  (:358-385).

trails instead has `packages/activesupport/src/time-ext.ts`: a module of
free functions taking a JS `Date` and returning `Temporal.Instant`, with a
`@boundary-file` header. It is not a reopening of any class, so the class
methods (`days_in_month`, `days_in_year`, `current`, `rfc3339`, `===`) and the
operator pairs have nowhere to land, and the instance methods lose their
receiver. `pnpm parity:api:extra` scores it 5 novel / 53 moved.

trails' `Time` analogue is `packages/date/src/time.ts` (`export class Time`,
:175), which today carries only `utc` (:183) and `strftime` (:264).

## Converged shape

Port `core_ext/time/calculations.rb` onto the `date` package's `Time` class
using the settled mixin idiom (CLAUDE.md "Module mixins"): `this`-typed
functions in a file at the Rails path —
`packages/activesupport/src/core-ext/time/calculations.ts` — assigned onto
`Time` where the two packages already meet, so the code lives at the Rails
name in the Rails-shaped file. Keep the Rails method names, parameter names,
defaults and control flow.

Open design question for triage: activesupport depends on `date`, so the
assignment site (an activesupport-side `include()`/static-assignment vs. a
`date`-side hook) needs to be settled before the first slice lands. Confirm
which direction the existing package graph allows.

**When this lands, update the override.** #6160 pinned
`"activesupport:core_ext/time/calculations.rb": "time-ext.ts"` in
`RUBY_FILE_TS_OVERRIDES` (`scripts/parity/conventions.ts`). A port to
`core-ext/time/calculations.ts` makes that entry wrong — delete it so the
default kebab-case rule resolves the file, and the entry's shared comment
block (it covers all three calculations.rb files) shrinks accordingly.

Related: `activesupport-core-ext-calculations-delegation` covers the
call-set divergences _within_ time-ext.ts; if this story lands first it
subsumes the `time-ext.json` baseline rows for `Time`, which must then be
deleted by hand (the baseline is only-shrink — never `--write`/reseed).

## Acceptance criteria

- `core_ext/time/calculations.rb` methods live on trails' `Time` at the Rails
  names, in a Rails-shaped file, with Rails control flow and decomposition.
- `RUBY_FILE_TS_OVERRIDES` entry for this file removed; `pnpm parity:api`
  activesupport ported-method count strictly up.
- Any `time-ext.json` call-mismatch rows the port retires are deleted by hand.
- `pnpm parity:api:calls`, `pnpm parity:api:extra` green; no new `@noRailsEquivalent`.
- Rails' own test names, verbatim, from
  `vendor/rails/activesupport/test/core_ext/time_ext_test.rb`.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0113-branch-and-guard-parity`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
