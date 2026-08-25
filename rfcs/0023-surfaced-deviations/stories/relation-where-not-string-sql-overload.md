---
title: "Relation#whereNot / WhereChain#not accept no bare SQL string, unlike Rails' where.not"
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

Rails' `WhereChain#not` accepts a bare SQL string:

```ruby
User.where.not("name = 'Jon'")
# SELECT * FROM users WHERE NOT (name = 'Jon')
```

documented at `vendor/rails/activerecord/lib/active_record/relation/query_methods.rb:22-27`
and implemented by `not(opts, *rest)` forwarding straight to
`build_where_clause(opts, rest).invert` (query_methods.rb:49-55), which handles
the string form exactly as `where` does.

trails cannot express this. `WhereChain#not`
(`packages/activerecord/src/relation/query-methods.ts:66-78`) delegates to
`Relation#whereNot`, whose overloads
(`packages/activerecord/src/relation.ts:802-804`) are only:

- `whereNot(conditions: Record<string, unknown>)`
- `whereNot(conditions: unknown[])`
- `whereNot(cols: string[], tuples: unknown[][])`

There is no `whereNot(sql: string, ...binds: unknown[])`, even though
`Relation#where` has exactly that overload (`relation.ts:546`). So the
sanitized-array workaround `where().not(["name = ?", x])` works but the direct
`where().not("name = ?", x)` — which Rails supports — does not.

Surfaced while converging `Base.whereNot` onto the `where().not` chain
(PR #5921, story `converge-base-where-not-to-where-chain`); that story's scope
explicitly excluded touching `Relation#whereNot`, so the gap was left in place.

## Acceptance criteria

- `Relation#whereNot` gains the `whereNot(sql: string, ...binds: unknown[])`
  overload, routing through the same `buildWhereClause` path `Relation#where`
  uses for its string form, then inverting — matching Rails'
  `build_where_clause(opts, rest).invert`.
- `WhereChain#not` gains the matching string overload so
  `Model.where().not("name = ?", x)` works.
- Port the Rails coverage for the string form rather than inventing test names:
  check `vendor/rails/activerecord/test/cases/relation/where_test.rb` and
  `relation/where_chain_test.rb` for the `where.not` string cases and mirror
  those test names verbatim.
- Disambiguation must not regress the existing single-array
  sanitized-conditions form or the composite `(cols, tuples)` form — both are
  covered in `packages/activerecord/src/relation/composite-where.test.ts`.
