---
title: "sqlite3 explain forwards binds where Rails passes []"
status: ready
updated: 2026-08-25
rfc: "0119-connection-adapter-fidelity"
cluster: null
deps: []
deps-rfc: []
est-loc: 40
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Rails' sqlite3 `explain` passes **no** binds to the EXPLAIN
(`vendor/rails/activerecord/lib/active_record/connection_adapters/sqlite3/database_statements.rb:18-21`):

```ruby
sql    = "EXPLAIN QUERY PLAN " + to_sql(arel, binds)
result = internal_exec_query(sql, "EXPLAIN", [])
```

It can, because `to_sql(arel, binds)` has already produced a bind-free
statement by that point. Trails' `AbstractSQLite3Adapter#explain`
(`packages/activerecord/src/connection-adapters/sqlite3-adapter.ts:949`)
forwards `binds`, because `Relation#execExplain`
(`packages/activerecord/src/relation.ts:3018`) collects real `[sql, binds]`
pairs off the `sql.active_record` stream — the SQL still carries `?`
placeholders, so EXPLAIN'ing it with `[]` would leave the parameters unbound.

So the deviation is upstream of `explain`: trails' collected-query
representation is bind-carrying where Rails' `to_sql` output is not. Noted and
accepted during review of #5744; tracked here so it converges rather than
silently persisting.

## Acceptance criteria

- sqlite3 `explain` renders the collected sql+binds to a bind-free statement
  and passes `[]` to `internalExecQuery`, matching Rails' `to_sql(arel, binds)`
  shape verbatim.
- `explain.test.ts` and `adapters/sqlite3/explain.test.ts` pass on sqlite.
- If `Relation#explain`'s bind rendering genuinely blocks this (it needs the
  binds for its header — `_renderExplainBinds`), that dependency is the
  blocker: `pnpm tasks block` naming it, or file the `_renderExplainBinds`
  prerequisite as its own story first. Do NOT close this as a justified
  deviation — a deviation-convergence story always converges (CLAUDE.md).
