---
title: "Retire inQueryConnection: Rails' calculate opens no lease, execute_*_calculation do"
status: draft
updated: 2026-08-15
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 180
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while converging the internal query leases onto `with_connection`
(PR #6570, RFC 0106).

`packages/activerecord/src/relation/calculations.ts` wraps every public
calculation entry point in a trails-invented `inQueryConnection` decorator:

```ts
export const Calculations = {
  calculate: inQueryConnection(calculate),
  count: inQueryConnection(performCount),
  sum: inQueryConnection(performSum),
  ...
};
```

It does two things Rails does not:

1. It opens `model.with_connection` around the WHOLE calculation. Rails' public
   `calculate` (`vendor/rails/activerecord/lib/active_record/relation/calculations.rb:200-231`)
   opens no lease at all — the lease is opened by
   `execute_simple_calculation` (:496-499) and `execute_grouped_calculation`
   (:527), each around its own `select_all`, which PR #6570 already converged.
   The outer wrap is now redundant: it is a second, wider lease that no Rails
   body has, and it is what makes `calculate`'s call set carry a
   `with_connection` Rails does not make.
2. It runs `_materializeDeferredDistinctPkPredicates()` — a trails-only step
   with no Rails counterpart — inside that wrap, before dispatching.

## Converged shape

Delete `inQueryConnection` and assign the calculation functions to the
`Calculations` map directly, as Rails' module does. The leases the queries
actually need are already in the two `execute_*_calculation` bodies and in
`select_for_count` (`calculations.rb:645-653`), all converged by #6570.

The deferred distinct-PK materialization has to keep running before the where
clause compiles; the settled place for it is the body that compiles the arel
(the way `ids` now runs it against the relation it compiles,
`relation.ts` `ids()`), not a decorator around the public surface. Check
whether it can move into `execute_simple_calculation` /
`execute_grouped_calculation` next to their own `with_connection`, or whether
it belongs at `.where()`-build time (which is where Rails materializes the
equivalent, per the existing comment at the call site).

Related: [[calculations-aggregate-funnel-inversion]] converges the funnel these
wrappers sit on top of; do that one first if both are scheduled, since it moves
the bodies this decorator wraps.

## Acceptance criteria

- [ ] No `inQueryConnection` in `relation/calculations.ts`; the `Calculations`
      map holds the ported functions directly.
- [ ] The deferred distinct-PK materialization runs from a body that has a
      Rails counterpart, not from a decorator.
- [ ] No new call-SET or call-ARG rows; the `calculate` / `count` / `sum` /
      `average` / `minimum` / `maximum` bodies still hold their leases where
      Rails holds them.
- [ ] SQLite, PG and MySQL/MariaDB green — the redundant outer lease is only
      observable under a real pool.
