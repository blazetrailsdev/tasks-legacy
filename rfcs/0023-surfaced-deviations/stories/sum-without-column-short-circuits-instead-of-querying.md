---
title: "sum-without-column-short-circuits-instead-of-querying"
status: draft
updated: 2026-08-02
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 60
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while converging the calculation dispatch shims in #5897
(`converge-calculation-and-batch-dispatch-shim-bodies`).

Rails' `sum` takes `initial_value_or_column = 0` and always funnels into
`calculate` (`vendor/rails/activerecord/lib/active_record/relation/calculations.rb`,
`def sum(initial_value_or_column = 0, &block)`), so a bare `Model.sum` becomes
`calculate(:sum, 0)` and issues `SELECT SUM(0) FROM ...`.

trails' `performSum`
(`packages/activerecord/src/relation/calculations.ts`) instead short-circuits:

```ts
if (!column) return 0;
```

It returns `0` without issuing any SQL. The scalar answer agrees for the common
case (`SUM(0)` over N rows is 0), so this is invisible to value assertions, but
it diverges in two observable ways: the query count differs (a Rails test
wrapping `assert_queries` around a bare `sum` would see one query, trails zero),
and the `initial_value_or_column` semantics are lost — Rails lets a caller pass
a non-zero initial value as the first positional argument.

## Acceptance criteria

- `performSum` mirrors Rails' `initial_value_or_column` parameter shape rather
  than treating a missing column as "return 0 immediately".
- A bare `sum()` issues the same query Rails issues instead of short-circuiting.
- Any test that relied on the zero-query short-circuit is checked against the
  corresponding Rails test rather than kept as-is.
- `pnpm parity:api` and `pnpm parity:test` deltas are non-negative.
