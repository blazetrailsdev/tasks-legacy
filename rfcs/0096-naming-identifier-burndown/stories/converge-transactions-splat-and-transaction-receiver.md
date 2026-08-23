---
title: "converge-transactions-splat-and-transaction-receiver"
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

Four rows in `packages/activerecord/src/transactions.ts` (`before_commit`,
`after_commit`, `after_rollback`, `with_transaction_returning_status`) are a1
findings — a different argument list, not a different identifier. Two causes:

- Rails' `*args` splat
  (`activerecord/lib/active_record/transactions.rb`) is modelled in trails as a
  single kwargs `options` object, so the forwarded argument list cannot line up.
- Rails calls `transaction` as a method on the connection; trails' `transaction()`
  is a free function that takes the model class as its leading argument.

## Acceptance criteria

1. The splat forwarding matches Rails' argument list, and `transaction` is
   reached the way Rails reaches it (or the free-function shape is shown to be a
   genuine TS language shortcoming and carries a `@missingRailsArgs <ruby_call>
— PERMANENT <reason>` receipt at each call site).
2. The four rows are gone from the convergeable population in
   `pnpm parity:api:calls:args:report`; no new `call-mismatches-exclude/` row.
3. `pnpm parity:api:calls` and `pnpm parity:api:calls:args` stay green.
