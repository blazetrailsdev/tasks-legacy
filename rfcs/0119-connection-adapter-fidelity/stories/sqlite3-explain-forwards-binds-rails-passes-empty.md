---
title: "sqlite3 explain forwards binds where Rails passes []"
status: draft
updated: 2026-07-31
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

- Determine whether trails' explain path can render the collected sql+binds to
  a bind-free statement before EXPLAIN (Rails' `to_sql(arel, binds)` shape).
- If it can, do so and pass `[]` to `internalExecQuery`, matching Rails
  verbatim.
- If it cannot without breaking `Relation#explain` bind rendering (which needs
  the binds for its header — `_renderExplainBinds`), record the reason at the
  call site and close this out as a justified deviation.
- `explain.test.ts` and `adapters/sqlite3/explain.test.ts` pass on sqlite.
