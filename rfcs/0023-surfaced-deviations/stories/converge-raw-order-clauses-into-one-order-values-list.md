---
title: "Delete _rawOrderClauses — raw order SQL belongs in _orderClauses as SqlLiteral (query_methods.rb:2015-2036)"
status: draft
updated: 2026-08-16
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
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

Rails keeps ONE order list. `order!` / `reorder!` push everything —
Arel::Nodes::Ordering, Arel::Attribute, Symbol, String, Hash — into
`order_values` (`query_methods.rb:1502-1505` for `reverse_order!`, `:2015-2036`
for `reverse_sql_order`), and every consumer reads that single list:
`build_arel`'s `arel.order(*order_values)`, `reverse_sql_order`'s
`order_query.empty?` default branch, `has_order?`, `ordered_relation`.

trails splits it in two. `packages/activerecord/src/relation.ts:293` declares
`private _rawOrderClauses: string[] = []` alongside `_orderClauses`, described
at `:5011` as "the trails-side carrier for order SQL Rails keeps in
`order_values`". Consumers must then check BOTH lists, and they do so
inconsistently — a partial census:

- `relation/finder-methods.ts` `hasOrder` — checks both (correct).
- `relation/query-methods.ts` `reverseOrderBang` — did NOT touch the raw list
  until PR #6599, which added the reversal and the empty-order-default
  suppression.
- `associations/disable-joins-association-scope.ts:212,318`,
  `disable-joins-association-relation.ts:328-354`,
  `associations/association-scope.ts:925-951`,
  `associations/preloader/through-association.ts:449-452` — each re-derives
  its own merge/guard over the pair.

Every one of those is a place where a single `order_values` list would need no
special case. The split is the root cause: PR #6599 deleted
`hasReversibleOrder`/`orderByPk` from `finder-methods.ts` (a caller-side
"is this order reversible" branch that existed only because the raw list was
invisible to `reverse_order!`), but the carrier itself survived.

## Converged shape

- `_rawOrderClauses` is deleted; raw order SQL enters `_orderClauses` as an
  `Arel::Nodes::SqlLiteral`, which is what Rails' `order_values` holds for a
  String order arg (`SqlLiteral < String` in Ruby, which is exactly why
  `reverse_sql_order`'s `when String` branch catches it).
- `reverseOrderBang` loses its dual handling; `hasOrder`, `orderedRelation`,
  `buildOrder` and the association-scope merges each collapse to the single
  list.
- The association scope/relation merge sites above lose their paired
  raw/structured branches.

Sequence this AFTER a census of the ~8 call sites — the risk is a consumer
that today reads only `_orderClauses` and would start seeing raw SQL it
previously never got.

## Acceptance criteria

- [ ] `_rawOrderClauses` no longer exists; raw order SQL lives in
      `_orderClauses` as SqlLiteral, per `query_methods.rb:2015-2036`.
- [ ] Every consumer listed above reads one list, with no raw/structured pair.
- [ ] `inOrderOf`, `last`/`last(n)` on raw-ordered relations, disable-joins
      association scopes and through-preloading stay green on SQLite,
      PostgreSQL and MySQL/MariaDB.
