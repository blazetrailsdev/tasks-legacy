---
title: "Route remaining where_clause writes through WhereClause#plus instead of in-place predicate pushes"
status: ready
updated: 2026-08-25
rfc: "0113-branch-and-guard-parity"
cluster: invented-arm
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

Surfaced while porting `WhereClause#+ #- #| #==` (PR #6084). That PR converged
the five call sites Rails writes as `where_clause +=`
(`activerecord/lib/active_record/relation/query_methods.rb:52, :1044, :1068`)
onto the new `WhereClause#plus`. The remaining sites still push into
`predicates` in place, which Rails never does — every Rails write goes through
`+`, `-`, `|` or `self.where_clause =`:

- `packages/activerecord/src/relation.ts:615` (`whereBang` node path)
- `packages/activerecord/src/relation.ts:916` (`or` / `and` combined ast)
- `packages/activerecord/src/relation.ts:1214`
- `packages/activerecord/src/relation.ts:4622,4625,4629` (`inBatches` range bounds)
- `packages/activerecord/src/relation/query-methods.ts:1278` (`none!` —
  Rails `query_methods.rb:1589` `self.where_clause += Relation::WhereClause.new(predicates)`)
- `packages/activerecord/src/relation/query-methods.ts:1470, :1495`

In-place mutation is also unsound where the clause array is shared with the
relation the caller cloned from; `plus` returns a new clause, as Ruby `Array#+`
does.

Also in the same file: `WhereClause#invert`
(`relation/where_clause.rb:83-91`) has two branches (1 predicate → invert it,
else `Not(ast)`), while `packages/activerecord/src/relation/where-clause.ts`
adds a third `predicates.length === 0` early return Rails does not have.

## Acceptance criteria

- Every `_whereClause.predicates.push(...)` listed above writes through the
  ported operator its Rails counterpart uses (`plus` for `+=`, assignment for
  `self.where_clause =`), matching the Rails line cited at each site.
- `WhereClause#invert` drops the extra empty-clause branch, or the branch is
  shown to be load-bearing and the reason cited at the call site with the Rails
  line that makes it necessary.
- No baseline row, `@noRailsEquivalent` tag or skip added.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0113-branch-and-guard-parity`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
