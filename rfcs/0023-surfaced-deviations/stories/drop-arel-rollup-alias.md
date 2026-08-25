---
title: "Drop the invented Rollup alias beside Rails' RollUp"
status: draft
updated: 2026-08-25
rfc: "0023-surfaced-deviations"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 30
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`packages/arel/src/nodes/unary.ts:56-59` exports a deprecated `Rollup` alias
(both a const and a type) beside the Rails-spelled `RollUp`, and
`packages/arel/src/nodes/index.ts:29` re-exports it. Rails has exactly one
spelling: `RollUp`, in the `Class.new(Unary)` list at
`vendor/rails/activerecord/lib/arel/nodes/unary.rb:25-42`, and the visitor
method is `visit_Arel_Nodes_RollUp`
(`vendor/rails/activerecord/lib/arel/visitors/postgresql.rb:54-57`). The alias
is invented public surface — it is part of arel's `parity:api:extra` total.

Surfaced while converging `postgres-visitor-assertion-parity` (PR #7013),
which rewrote the mirrored postgres test to the Rails spelling; the alias
survives only in trails-only tests.

## Converged shape

Delete the `Rollup` const and type from `unary.ts` and the `Rollup` entry from
`nodes/index.ts`; update the remaining `Nodes.Rollup` call sites
(`packages/arel/src/visitors/postgres.trails.test.ts:60,88` at the time of
writing — re-grep, they are trails-only tests) to `Nodes.RollUp`.

## Acceptance criteria

- No `Rollup` spelling remains in `packages/arel/src`.
- arel's `parity:api:extra` total drops by the alias' names; tighten the mark
  in `scripts/api-compare/extra-surface-mark.json` with
  `pnpm parity:api:extra:tighten` (only-shrink; never reseed).
- `pnpm vitest run packages/arel`; parity deltas non-negative.
