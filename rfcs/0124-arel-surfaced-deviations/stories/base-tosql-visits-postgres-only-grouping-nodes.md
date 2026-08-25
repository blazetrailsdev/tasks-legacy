---
title: "Base ToSql visits Cube/RollUp/GroupingElement/GroupingSet, which Rails leaves PostgreSQL-only"
status: draft
updated: 2026-08-25
rfc: "0124-arel-surfaced-deviations"
cluster: null
packages:
  - "arel"
deps: []
deps-rfc: []
est-loc: 70
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`packages/arel/src/visitors/to-sql.ts` defines base-visitor handlers for four
nodes Rails' base `ToSql` visitor does not visit at all:

- `visitArelNodesCube` (`CUBE(...)`)
- `visitArelNodesRollUp` (`ROLLUP(...)`)
- `visitArelNodesGroupingElement` (`(...)`)
- `visitArelNodesGroupingSet` (`GROUPING SETS(...)`)

`vendor/rails/activerecord/lib/arel/visitors/to_sql.rb` contains no
`visit_Arel_Nodes_Cube` / `RollUp` / `GroupingElement` / `GroupingSet` — grep
returns nothing. These are PostgreSQL-only in Rails:
`vendor/rails/activerecord/lib/arel/visitors/postgresql.rb:44-62` defines all
four, and `:88-96` is the `grouping_array_or_grouping_element` helper they
share. A non-PG visitor handed one of these nodes raises in Rails
(`Visitor#visit`'s `TypeError` terminal, `arel/visitors/visitor.rb:36-39`)
rather than emitting SQL.

Surfaced while converging `postgres-visitor-assertion-parity` (PR #7013),
which fixed the PG-side halves of the same cluster: `GroupingElement` no
longer coerces `expr` to an array, and the four node classes are now direct
`Unary` siblings (`arel/nodes/unary.rb:25-42`). The base-visitor handlers were
left in place because deleting them is a separate blast radius.

## Converged shape

Delete the four handlers from `to-sql.ts` and their `reg(...)` dispatch-cache
registrations (`to-sql.ts:471-475`), leaving `PostgreSQL` the only visitor that
answers them, exactly as `postgresql.rb` does. Check the arel and activerecord
suites for a test that compiles one of these nodes through a non-PG visitor —
that assertion is itself a divergence and should move to the PG visitor.

## Acceptance criteria

- No `Cube` / `RollUp` / `GroupingElement` / `GroupingSet` handler or dispatch
  registration remains in `to-sql.ts`.
- Compiling one through `Visitors::ToSql` raises the same `TypeError` any
  unhandled node raises.
- `pnpm vitest run packages/arel`; parity deltas non-negative.
