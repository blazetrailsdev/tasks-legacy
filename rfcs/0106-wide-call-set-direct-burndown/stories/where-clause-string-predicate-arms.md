---
title: "where-clause-string-predicate-arms"
status: claimed
updated: 2026-08-23
rfc: "0106-wide-call-set-direct-burndown"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: null
claim: "2026-08-23T12:42:25Z"
assignee: "where-clause-string-predicate-arms"
blocked-by: null
closed-reason: null
---

## Context

Split out of `query-methods-order-only-call-inversions` (RFC 0106): the one
row of that shard's four that could not converge in place.

`build_where_clause`'s `sanitize_sql` arm
(`packages/activerecord/src/relation/query-methods.ts:1092-1096`) wraps the
sanitized fragment in `new Nodes.SqlLiteral(...)`; Rails stores the bare String
(`activerecord/lib/active_record/relation/query_methods.rb:1627`,
`parts = [model.sanitize_sql(...)]`) and lets `Relation::WhereClause` handle the
String arm downstream — `where_clause.rb:160`, `167`, `190`, `203`
(`invert_predicate`, `predicates_with_wrapped_sql_literals`, `wrap_sql_literal`,
`non_empty_predicates`' `ARRAY_WITH_EMPTY_STRING`).

trails' `WhereClause` types `predicates` as `Nodes.Node[]` and has none of those
String arms, so dropping the pre-wrap here would push a raw String into a
clause that cannot handle it. The pre-wrap is what the call-set gate reports as
`build_where_clause order:constructor,buildFromHash` — a constructor Rails does
not make, ahead of `build_from_hash`.

## Acceptance criteria

- [ ] `WhereClause` carries Rails' String predicate arms
      (`where_clause.rb:160,167,190,203`) and its predicate type admits String.
- [ ] `build_where_clause`'s sanitize_sql arm is `parts = [model.sanitizeSql(...)]`,
      no `SqlLiteral` wrap.
- [ ] The `build_where_clause` / `order:constructor,buildFromHash` baseline row
      is deleted; `pnpm parity:api:calls` green.
