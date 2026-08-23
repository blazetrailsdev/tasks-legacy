---
title: "converge-activesupport-temporal-receiver-chaining"
status: ready
updated: 2026-08-23
rfc: "0096-naming-identifier-burndown"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

RFC 0096 wave-5 residual finding, split out of
`wave-5-residual-arg-shape-findings` (PR #6929).

Five arg-shape rows in activesupport come from the same cause: a chained
date/time calculation rebuilds a JS `Date` from a `Temporal.Instant` between
calls, where Ruby chains straight on the receiver.

- `core-ext/date-and-time/calculations.ts` — the private `receiver()`
  (calculations.ts:122-127) stands between every chained call, against
  `activesupport/lib/active_support/core_ext/date_and_time/calculations.rb:140`
  (`beginning_of_quarter`), `:155` (`end_of_quarter`), `:209` (`next_weekday`),
  `:236` (`prev_weekday`), each of which chains on `self`.
- `time-ext.ts#advance` — `since(new Date(timeAdvancedByDate.epochMilliseconds),
…)` where `core_ext/time/calculations.rb` passes `time_advanced_by_date`
  straight through.

## Acceptance criteria

1. The chained calls pass the receiver Rails passes, with no intermediate
   `Date` reconstruction.
2. The five rows are gone from `pnpm parity:api:calls:args:report`; no new
   `call-mismatches-exclude/` row.
3. `pnpm parity:api:calls` and `pnpm parity:api:calls:args` stay green, and the
   activesupport date/time suites pass on every adapter lane.
