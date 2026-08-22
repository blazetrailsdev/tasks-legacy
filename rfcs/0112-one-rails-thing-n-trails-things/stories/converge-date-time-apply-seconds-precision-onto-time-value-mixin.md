---
title: "Converge DateTimeType#applySecondsPrecision onto the Helpers::TimeValue mixin"
status: done
updated: 2026-08-22
rfc: "0112-one-rails-thing-n-trails-things"
cluster: dead-mixin-companions
packages: []
deps: []
deps-rfc: []
est-loc: 90
pr: 6738
claim: "2026-08-19T12:59:52Z"
assignee: "days-into-week-duplicated-in-date-calculations"
blocked-by: null
closed-reason: null
---

## Context

`vendor/rails/activemodel/lib/active_model/type/date_time.rb:47` is
`include Helpers::TimeValue`, so `DateTime#apply_seconds_precision` IS
`time_value.rb:24-34` — one implementation, shared with `Type::Time`.

trails does not mix it in. `packages/activemodel/src/type/date-time.ts` carries
a **private duplicate** (`applySecondsPrecision`, ~15 lines of bigint
epoch-nanosecond arithmetic) while the ported helper already exists at
`packages/activemodel/src/type/helpers/time-value.ts:53` and IS mixed into
`Type::Time` the settled way (`type/time.ts:26`,
`protected applySecondsPrecision = applySecondsPrecision`).

The two are not equivalent, which is why PR #6528 did not simply swap them:

- the helper truncates via Temporal's `.round({ roundingMode: "trunc" })`,
  which rounds **toward zero**;
- the date-time.ts copy floors — it normalizes a negative sub-second remainder
  up (`if (subsec < 0n) subsec += 1_000_000_000n`) before subtracting.

They therefore disagree for pre-1970 instants with sub-second precision. Ruby's
`value.change(nsec: value.nsec - rounded_off_nsec)` operates on `nsec`, which
is always non-negative, so **the floor behaviour is the Rails-correct one** and
the helper's `trunc` is the side that needs fixing.

`date-time.ts` also carries `_nsAtPrecision` (used by `isChanged`), a third
spelling of the same truncation with no Rails counterpart.

## Converged shape

One implementation: fix `helpers/time-value.ts#applySecondsPrecision` to floor
on the `nsec` field rather than `trunc` on the instant, then mix it into
`DateTimeType` as `type/time.ts:26` does and delete the private copy plus
`_nsAtPrecision`. `Type::Time` and `Type::DateTime` then share one method, as
`include Helpers::TimeValue` gives Rails.

## Acceptance criteria

- [ ] `DateTimeType` has no private `applySecondsPrecision` and no
      `_nsAtPrecision`; it mixes in `helpers/time-value.ts`'s function under
      the Rails name.
- [ ] A test pins a pre-1970 instant with sub-second precision through both
      `Type::Time` and `Type::DateTime` and shows they agree with
      `time_value.rb:24-34`'s `nsec` arithmetic.
- [ ] `pnpm parity:api:extra --package activemodel` loses the `_nsAtPrecision`
      row.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
