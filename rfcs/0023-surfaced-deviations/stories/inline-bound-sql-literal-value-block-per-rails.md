---
title: "Inline the bound-sql-literal value block into both Rails bodies"
status: draft
updated: 2026-08-16
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
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

`relation/query-methods.ts` extracts `normalizeBoundValue` — a `this`-typed
helper with no Rails counterpart — out of the bodies of
`build_bound_sql_literal` and `build_named_bound_sql_literal`. Rails does NOT
extract it: query_methods.rb:1682-1700 (`build_named_bound_sql_literal`) and
:1702-1720 (`build_bound_sql_literal`) each carry the same three-branch block
inline, once over `values.transform_values` and once over `values.map`:

```ruby
if ActiveRecord::Relation === value
  Arel.sql(value.to_sql)
elsif value.respond_to?(:map) && !value.acts_like?(:string)
  values = value.map { |v| v.respond_to?(:id_for_database) ? v.id_for_database : v }
  values.empty? ? nil : values
else
  value = value.id_for_database if value.respond_to?(:id_for_database)
  value
end
```

Surfaced during PR #6587 (RFC 0106 call-set burndown): the `sql` call-set rows
for both methods were retired there by the `Arel.sql` import rename, so this is
no longer a call-set row — it is invented surface (`pnpm parity:api:extra`) and
a decomposition divergence, which is why it was left out of that PR's scope.

The complication: a THIRD caller exists at `relation.ts` `updateAll`
(`_qm.normalizeBoundValue.call(this, v)` in the `[sql, *binds]` arm), where
Rails reaches `sanitize_sql_for_assignment` instead. Inlining the block into
the two Rails bodies without addressing that caller just moves the problem, so
the `updateAll` arm has to be converged in the same pass.

## Acceptance criteria

- [ ] The block is inlined into `buildBoundSqlLiteral` and
      `buildNamedBoundSqlLiteral`, matching Rails' duplication
      (query_methods.rb:1682-1720). One Rails method is one TS method.
- [ ] `relation.ts` `updateAll`'s `[sql, *binds]` arm no longer routes through
      the helper; check it against relation.rb `update_all`'s
      `sanitize_sql_for_assignment` path.
- [ ] `normalizeBoundValue` is gone from the exported surface;
      `pnpm parity:api:extra --package activerecord` shows one fewer novel name
      in `relation/query-methods.ts`.
- [ ] `pnpm parity:api:calls` / `:args` stay green with no new rows.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.
