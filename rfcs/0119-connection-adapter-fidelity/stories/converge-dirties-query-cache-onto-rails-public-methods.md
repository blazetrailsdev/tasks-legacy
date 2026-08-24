---
title: "dirties_query_cache is wired on exec_insert/update/delete, not Rails' public insert/update/delete — leaving PG's non-returning arm uncleared"
status: draft
updated: 2026-08-21
rfc: "0119-connection-adapter-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 140
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Rails wires `dirties_query_cache` onto the **public** statement methods
(`vendor/rails/activerecord/lib/active_record/connection_adapters/abstract/query_cache.rb:13-15`):

```ruby
dirties_query_cache base, :exec_query, :execute, :create, :insert, :update, :delete, :truncate,
  :truncate_tables, :rollback_to_savepoint, :rollback_db_transaction, :restart_db_transaction,
  :exec_insert_all
```

Note what is **not** in that list: `exec_insert`, `exec_update`, `exec_delete`.
trails wires those three instead of Rails' `insert` / `update` / `delete`
(`packages/activerecord/src/connection-adapters/abstract-adapter.ts`, the
`dirtiesQueryCache(AbstractAdapter, ...)` call at the bottom of
`ensureAbstractAdapterMixinsApplied`). The comment there justifies the lower
wiring: trails' model save path (`_updateRecord` in `persistence.ts`,
`_destroyRow` in `base.ts`) calls `execUpdate` / `execDelete` directly, bypassing
the public `update` / `delete` that Rails wires — so wiring Rails' set alone would
miss the save path.

### The concrete gap this leaves (surfaced by PR #6835)

PR #6835 converged `exec_insert` to Rails' shape and moved the `execInsert`
wiring from the three concrete adapters onto `AbstractAdapter`, because only
PostgreSQL still overrides `execInsert` and its override delegates to `super`
(`postgresql/database_statements.rb:46-47`) — wiring it on both levels clears the
cache twice for one logical insert and reds the exact-count assertions in
`query-cache.test.ts` (`assertClears(1, ...)`).

The cost is that PostgreSQL's **else** arm — the `use_insert_returning? == false`
path at `postgresql/database_statements.rb:48-59` — no longer clears the query
cache at all: it calls `internalExecQuery` directly, which is permanently
unwrapped by design, and it sits above the now-wired `AbstractAdapter#execInsert`
rather than delegating through it.

That arm is unreachable in trails today (nothing sets `insert_returning: false`,
so `_useInsertReturning` is always true), which is why no test caught it. It is
still a real hole, and Rails does not have it: Rails clears on the public
`insert`, which is above **both** arms.

## Converged shape

Wire the Rails set from `query_cache.rb:13-15` — `insert` / `create` / `update` /
`delete` rather than `execInsert` / `execUpdate` / `execDelete` — so the clear
happens above every adapter's arm-splitting override, exactly once per logical
write, and PostgreSQL's non-returning arm is covered by construction.

That requires first closing the reason trails wired lower: the save path must
reach the public `update` / `delete`, or Rails' own save path must be shown to
bypass them too (check `_update_record` / `_delete_row` in
`vendor/rails/activerecord/lib/active_record/persistence.rb` — Rails' go through
`connection.update(arel, name)` / `connection.delete(...)`, which ARE the wired
public methods). If the trails save path is what diverges, converging it is the
prerequisite and belongs in this story or a split of it.

## Acceptance criteria

- [ ] `dirtiesQueryCache` is wired on the Rails method set from
      `query_cache.rb:13-15`; `execInsert` / `execUpdate` / `execDelete` are no
      longer wired.
- [ ] PostgreSQL's `use_insert_returning? == false` arm clears the query cache.
- [ ] Each logical write still clears exactly once — `query-cache.test.ts`'s
      `assertClears(1, ...)` cases stay green on SQLite, PostgreSQL and
      MySQL/MariaDB (a double-clear is what forced the current wiring).
- [ ] A regression test drives the PostgreSQL non-returning arm with
      `insert_returning: false` and asserts one clear. It must fail on baseline.
