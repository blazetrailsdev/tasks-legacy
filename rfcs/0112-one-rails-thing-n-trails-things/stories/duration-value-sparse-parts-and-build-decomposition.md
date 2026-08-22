---
title: "Duration: port @value, zero-rejected sparse parts, and build decomposition"
status: done
updated: 2026-08-22
rfc: "0112-one-rails-thing-n-trails-things"
cluster: duplicate-bodies
deps: []
deps-rfc: []
est-loc: 200
pr: 6777
claim: "2026-08-20T17:22:15Z"
assignee: "converge-event-children-invention"
blocked-by: null
closed-reason: null
---

## Context

PR #5105 ported `@variable` tracking into trails Duration
(packages/activesupport/src/duration.ts) but left two structural deviations
from Rails `ActiveSupport::Duration` (vendor/rails/activesupport/lib/
active_support/duration.rb):

- No `@value` field: Rails' `initialize(value, parts, variable)` carries total
  seconds separately (`attr_reader :value`, delegations to it at
  duration.rb:224); trails derives totals via `inSeconds()` from a full fixed
  `parts` record, and the constructor takes `(parts, variable)` only.
- `parts` is a full record, never zero-rejected: Rails does
  `@parts.reject! { |k, v| v.zero? } unless value == 0` (duration.rb:228) and
  `parts` returns a sparse dup; trails always returns all seven keys.
- `Duration.build` (duration.ts) does NOT decompose seconds into calendar
  parts — Rails build (duration.rb:191-214) splits into years/months/… and
  computes `variable` from the nonzero parts; trails returns a seconds-only
  Duration. Consequently `modulo` (Rails `%` goes through `build`,
  duration.rb:311-319) also returns a seconds-only Duration.

## Update from #6465

Two notes from the PR that ported `Duration#_parts`:

- The **sparse-parts half of this story is now done**: `_parts()` returns the
  zero-rejected set via `_partKeys`/`_transformValues`, matching
  `@parts.reject! { |k, v| v.zero? }` (`duration.rb:228`). That PR also retired
  `_givenParts`, a trails-invented private that was Rails' `_parts` under
  another name. The `@value` field and `Duration.build` decomposition remain.
- The missing `@value` seat **cost a call-mismatch baseline row**:
  `scripts/api-compare/call-mismatches-exclude/activesupport/duration.json`
  now suppresses `parse -> calculate_total_seconds`, because Rails' `parse` is
  `new(calculate_total_seconds(parts), parts)` (`duration.rb:212-215`) and
  trails' `(parts, variable)` constructor gives that call no argument to fill.
  **Delete that row as part of this story** once `value` exists — it is the
  concrete debt this convergence retires.

## Acceptance criteria

- Duration carries `value` and zero-rejected sparse parts matching Rails'
  constructor shape, or the deviation is justified at the call sites and
  excluded coherently.
- `Duration.build` decomposes per duration.rb:191-214 (PARTS_IN_SECONDS
  div/mod loop) and `parts`/`variable` match Rails for built durations;
  `modulo` inherits the fix via build.
- DurationTest ports covering `build`/`parts` pass unchanged names.

## Update from the 2026-08-18 RFC 0023 triage pass

Two of the three deviations are now converged in `packages/activesupport/src/duration.ts`:

- **`@value` — DONE.** `readonly value: number` exists at duration.ts:102 with the
  `attr_reader :value` cite (duration.rb:133), and the constructor is
  `constructor(value, parts = {}, variable = null)` (duration.ts:106), matching
  Rails' `initialize(value, parts, variable = nil)`. Every factory
  (`Duration.seconds`/`minutes`/`hours`/…) fills it through that seat.
- **Sparse `parts` — DONE** (already recorded above from #6465); `_partKeys` at
  duration.ts:122 does the `value == 0` guard plus the zero-rejection.
  `_givenParts` and `_transformValues` no longer exist repo-wide.

**Only the `build` decomposition remains.** `Duration.build`
(duration.ts:618-624) is still:

```ts
static build(value: unknown): Duration {
  if (typeof value !== "number") { /* TypeError */ }
  return new Duration(value, { seconds: value });
}
```

Rails' `build` (duration.rb:191-214) splits the seconds into
years/months/weeks/days/hours/minutes/seconds and derives `variable` from the
nonzero parts. Because Rails' `%` goes through `build`
(duration.rb:311-319), `modulo` inherits the seconds-only shape here too.

Scope this story to that one method plus `modulo`.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
