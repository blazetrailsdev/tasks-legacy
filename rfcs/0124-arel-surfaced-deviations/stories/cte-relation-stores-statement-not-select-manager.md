---
title: "Store a SelectManager in Cte.relation so visit_Arel_Nodes_Cte drops its invented paren branch"
status: draft
updated: 2026-08-25
rfc: "0124-arel-surfaced-deviations"
cluster: null
packages:
  - "arel"
deps: []
deps-rfc: []
est-loc: 120
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while folding `predications-range.ts` into `predications.ts` (#6856).

Rails' `visit_Arel_Nodes_Cte` (`activerecord/lib/arel/visitors/to_sql.rb:732-744`)
emits only the name, `" AS "`, the optional MATERIALIZED modifier, and then
`visit o.relation, collector` — **no parentheses at all**. It can do that because
`Cte#relation` holds a `SelectManager`, and `visit_Arel_SelectManager`
(`to_sql.rb:358-361`) emits its own `(...)`.

trails stores a bare `SelectStatement` / `SqlLiteral` in `Cte.relation`, which
does not self-wrap, so both visitor ports carry an invented paren decision:

- `packages/arel/src/visitors/to-sql.ts` `visitArelNodesCte` (~:1240)
- `packages/arel/src/visitors/mysql.ts` `visitArelNodesCte` (~:126), mirroring
  `activerecord/lib/arel/visitors/mysql.rb:72-76`

Each now inlines a six-term `instanceof` chain (`SelectManager`, `Grouping`,
`Union`, `UnionAll`, `Intersect`, `Except`) guarding an explicit `(` / `)`. #6856
inlined the previously-shared `cteRelationSelfWraps` helper because Rails
redefines the method in `mysql.rb` rather than sharing a helper — but the
duplicated _condition_ is the symptom, not the disease.

## Converged shape

`Cte.relation` holds a `SelectManager` (Rails' own CTE idiom is
`Arel::Nodes::As.new(cte_table, select_manager)`), so both
`visitArelNodesCte` bodies reduce to Rails' four lines with no paren branch and
no `instanceof` chain at all. Check the `Cte` constructor
(`packages/arel/src/nodes/cte.ts`) and every producer — notably the array-CTE
(`UnionAll`) and `SqlLiteral` paths the current comment calls out — before
changing the stored type.

## Acceptance criteria

- `visitArelNodesCte` in both `to-sql.ts` and `mysql.ts` matches
  `to_sql.rb:732-744` / `mysql.rb:72-76` line for line, with no paren branch.
- The six-term `instanceof` chain is gone from both files.
- `pnpm vitest run packages/arel` green, in particular the CTE / array-CTE /
  `SqlLiteral`-CTE cases (`AS ((…))` double-wrap is the regression to watch).
- `pnpm parity:api:calls` clean for both bodies.
