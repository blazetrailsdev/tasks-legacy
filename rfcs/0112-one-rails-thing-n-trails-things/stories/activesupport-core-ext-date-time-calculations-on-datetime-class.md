---
title: "Port core_ext/date_time/calculations.rb onto the DateTime class, not free functions"
status: in-progress
updated: 2026-08-22
rfc: "0112-one-rails-thing-n-trails-things"
cluster: dead-mixin-companions
deps: []
deps-rfc: []
est-loc: 350
pr: 6891
claim: "2026-08-22T23:00:03Z"
assignee: "activesupport-core-ext-date-time-calculations-on-datetime-class"
blocked-by: null
closed-reason: null
---

## Context

Surfaced by #6160 (`api-compare-orphan-buckets-activesupport-calculations`),
which pointed `activesupport:core_ext/date_time/calculations.rb` at
`packages/activesupport/src/time-ext.ts` and made its 38 methods visible for
the first time. **12 match, 25 still read missing.**

The deviation is one of shape, not of coverage. Rails reopens the `DateTime`
class:

- `vendor/rails/activesupport/lib/active_support/core_ext/date_time/calculations.rb:5`
  — `class DateTime`, with class method `current` (:10) and instance methods
  `seconds_since_midnight` (:20), `seconds_until_end_of_day` (:29), `subsec`
  (:36), `change` (:51), `advance` (:82), `ago` (:109), `since` (:116),
  `beginning_of_day` (:122), `middle_of_day` (:130), `end_of_day` (:140),
  `beginning_of_hour` (:146), `end_of_hour` (:152), `beginning_of_minute`
  (:158), `end_of_minute` (:164), `localtime` (:170), `utc` (:184), `utc?`
  (:197), `utc_offset` (:202), `<=>` (:208), plus the `getlocal`/`getgm`/
  `getutc`/`gmtime` aliases.

trails instead has `packages/activesupport/src/time-ext.ts`: free functions
over JS `Date` returning `Temporal.*`. The whole offset-handling cluster —
`localtime`/`getlocal`/`utc`/`getgm`/`getutc`/`gmtime`/`utc?`/`utc_offset` —
has no home there, and `<=>` and `subsec` are absent.

trails' `DateTime` analogue is `packages/date/src/date.ts`
(`export class DateTime extends Date`, :2476).

## Converged shape

Port `core_ext/date_time/calculations.rb` onto the `date` package's `DateTime`
class using the settled mixin idiom (CLAUDE.md "Module mixins"): `this`-typed
functions in a file at the Rails path —
`packages/activesupport/src/core-ext/date-time/calculations.ts` — assigned onto
`DateTime`, so the code lives at the Rails name in the Rails-shaped file. Keep
the Rails method names, parameter names, defaults and control flow, and port
the aliases as aliases rather than as re-implementations.

`DateTime` already carries its offset in trails (#6151), so the
`localtime`/`utc_offset` arm has a real receiver to work against — this slice
should not need new offset machinery.

**When this lands, update the override.** #6160 pinned
`"activesupport:core_ext/date_time/calculations.rb": "time-ext.ts"` in
`RUBY_FILE_TS_OVERRIDES` (`scripts/api-compare/conventions.ts`). A port to
`core-ext/date-time/calculations.ts` makes that entry wrong — delete it so the
default kebab-case rule resolves the file.

Related: `activesupport-core-ext-calculations-delegation` covers the call-set
divergences _within_ time-ext.ts; rows it shares with this port must be
deleted from `time-ext.json` by hand (the baseline is only-shrink — never
`--write`/reseed).

## Acceptance criteria

- `core_ext/date_time/calculations.rb` methods live on trails' `DateTime` at
  the Rails names, in a Rails-shaped file, with Rails control flow and
  decomposition; aliases ported as aliases.
- `RUBY_FILE_TS_OVERRIDES` entry for this file removed; `pnpm parity:api`
  activesupport ported-method count strictly up.
- `pnpm parity:api:calls`, `pnpm parity:api:extra` green; no new `@noRailsEquivalent`.
- Rails' own test names, verbatim, from
  `vendor/rails/activesupport/test/core_ext/date_time_ext_test.rb`.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
