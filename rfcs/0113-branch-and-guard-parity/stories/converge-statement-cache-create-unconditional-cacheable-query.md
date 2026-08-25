---
title: "StatementCache.create calls cacheable_query unconditionally, dropping the invented fallback arm"
status: ready
updated: 2026-08-25
rfc: "0113-branch-and-guard-parity"
cluster: invented-arm
packages: []
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

Surfaced by PR #6619 (RFC 0096 wave-4 naming burndown), which reported the row
`statement-cache.ts` / `create` / `new` (`ref:binds` -> `ref:sql`) as recorder
shape. It is not: the row exists because trails added a branch Rails does not
have.

Rails (`vendor/rails/activerecord/lib/active_record/statement_cache.rb:132-137`):

```ruby
def self.create(connection, callable = nil, &block)
  relation = (callable || block).call Params.new
  query_builder, binds = connection.cacheable_query(self, relation.arel)
  bind_map = BindMap.new(binds)
  new(query_builder, bind_map, relation.model)
end
```

`cacheable_query` is unconditional — every adapter has it
(`abstract/database_statements.rb:56-64`), and so does trails:
`packages/activerecord/src/connection-adapters/abstract/database-statements.ts:382`
defines it and `abstract-adapter.ts:634` exposes it on the base adapter.

trails (`packages/activerecord/src/statement-cache.ts:194-219`) nonetheless
types the parameter as `cacheableQuery?(...)` — optional — and carries an else
arm that builds `new Query(connection.toSql(arel))` with `binds = []`. That arm
is unreachable through any real adapter, and it is what the recorder pairs
against Rails' `BindMap.new(binds)`.

## Acceptance criteria

- [ ] `StatementCache.create` calls `connection.cacheableQuery(StatementCache,
relation.arel)` unconditionally and the `cacheableQuery?` optionality is
      gone from the parameter type.
- [ ] The dead else arm (`new Query(sql)` / `binds = []`) is deleted, not
      guarded.
- [ ] The `statement-cache.ts` / `create` / `new` naming row clears in
      `pnpm parity:api:calls:args:report`, with no new `shape` row.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0113-branch-and-guard-parity`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
