---
title: "Converge time-ext.ts's seven surfaced call-parity rows by moving the Time arm onto a gem-shaped receiver"
status: draft
updated: 2026-08-17
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activesupport"
deps: []
deps-rfc: []
est-loc: 300
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

PR #6635 ported `Time::DATE_FORMATS` into
`packages/activesupport/src/time-ext.ts`. That brought
`core_ext/time/conversions.rb` into the compared population for that TS file,
which surfaced **seven pre-existing divergences** in bodies #6635 did not
touch. They were baselined (each with its own reason, not a seed string) in
`scripts/api-compare/call-mismatches-exclude/activesupport/time-ext.json`.
Baseline rows are debt, not permission — this story converges them.

Four are one class: a Ruby zero-arg send to the receiver, ported as a free
function taking the receiver first.

- `change` → `integer?` (`core_ext/time/calculations.rb:154`,
  `new_time.utc_offset.integer?`) — TS passes `ref:offsetNanoseconds`.
- `preserve_timezone` → `system_local_time?()`
  (`core_ext/time/compatibility.rb`) — TS passes `ref:time`.
- `to_time` → `preserve_timezone()`, twice
  (`core_ext/time/compatibility.rb` and `core_ext/date_time/compatibility.rb`)
  — TS passes `ref:receiver`.

The remaining three:

- `change` → `local` (`core_ext/time/calculations.rb:172`):
  `::Time.local(new_sec, new_min, new_hour, new_day, new_month, new_year, nil,
nil, isdst, nil)` is the ten-argument form; `time-ext.ts`'s `local` helper is
  the six-field civil form and drops `usec`/`yday`/`isdst`/`tz`.
- `change` → `new` (`core_ext/time/calculations.rb:139` vs `:148`): a false
  pairing — the TS `new` the extractor sees is
  `new ArgumentError("argument out of range")`, while Rails' `::Time.new` arms
  are spelled `Temporal.ZonedDateTime.from`, which carries no `new`. Converging
  this one may mean teaching the extractor, not the body.
- `change` → `order:local,isInteger`: `time-ext.ts` runs Rails' `elsif zone`
  arm (`:172-173`) BEFORE the `elsif zone.respond_to?(:utc_to_local)` arm
  (`:150-171`), because a JS `Date` carries no zone object and is dispatched by
  receiver type up front.

## Converged shape

The root cause of six of the seven is that `time-ext.ts` models Ruby's `Time`
as a JS `Date`, so no member can be a method on the receiver and Ruby's branch
order cannot survive the receiver-type dispatch. The real convergence is to
move the `Time` arm onto a gem-shaped receiver — `@blazetrails/date`'s `Time`,
which already carries `utcOffset`, `zone` and `strftime` — the way
`core-ext/date-time/calculations.ts` and `.../conversions.ts` already do for
`DateTime`. Then `integer?`, `preserve_timezone` and `system_local_time?`
become receiver sends, `change`'s branches recover Rails' order, and the
ten-argument `Time.local` has a receiver to build from.

Rails anchors: `core_ext/time/calculations.rb:123-175`,
`core_ext/time/compatibility.rb`, `core_ext/date_time/compatibility.rb`.

## Acceptance criteria

- [ ] Each converged row is DELETED from
      `call-mismatches-exclude/activesupport/time-ext.json` by hand (only-shrink;
      never `--write`/reseed), and the stale high-water mark retired with
      `pnpm parity:api:calls:tighten activesupport/time-ext.json`.
- [ ] `pnpm parity:api:calls` / `:args` clean; `parity:api` / `parity:test`
      deltas non-negative.
- [ ] No behaviour change: the activesupport time/date suites stay green.
- [ ] If a row genuinely cannot converge, `pnpm tasks block` with the specific
      blocker — do not rewrite its reason in place.
