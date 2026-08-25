---
title: "StatementCache#execute drops Rails' async: arm (async_find_by_sql / Promise.wrap)"
status: draft
updated: 2026-08-22
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: ["port-promise-complete-for-async-loaded-arms"]
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

Rails' `StatementCache#execute` takes an `async:` kwarg and branches on it:

```ruby
def execute(params, connection, allow_retry: false, async: false, &block)
  bind_values = @bind_map.bind params
  sql = @query_builder.sql_for bind_values, connection

  if async
    @model.async_find_by_sql(sql, bind_values, preparable: true, allow_retry: allow_retry, &block)
  else
    @model.find_by_sql(sql, bind_values, preparable: true, allow_retry: allow_retry, &block)
  end
rescue ::RangeError
  async ? Promise.wrap([]) : []
end
```

(`activerecord/lib/active_record/statement_cache.rb:144-155`).

trails' `execute` (`packages/activerecord/src/statement-cache.ts:227-254`) has no
`async` parameter at all: it always calls `findBySql` and always returns `[]`
from the `RangeError` rescue. Two call-set baseline rows were left with per-site
reasons for this in PR #6845 (`statement-cache.json`, calls `async_find_by_sql`
and `wrap`).

The reason it was not converged there: **no caller can pass `async: true` yet.**
Rails reaches it from `Association#find_target`:

```ruby
klass.with_connection do |c|
  sc.execute(binds, c, async: async) do |record|
    set_inverse_instance(record)
    set_strict_loading(record)
  end
end
```

(`activerecord/lib/active_record/associations/association.rb:267-273`), and the
`async` local there comes from the `load_async` path, which trails does not model
— `ActiveRecord::Promise` (`activerecord/lib/active_record/promise.rb`) is
unported. Adding the arm today would be an unreachable code path, and
`Promise.wrap([])` has nothing to construct.

This story therefore **depends on** `port-promise-complete-for-async-loaded-arms`
(0023), which ports `Promise::Complete` and the `@async` arms that currently drop
it. Do that first.

Note `asyncFindBySql` already exists (`packages/activerecord/src/querying.ts:68`)
but is currently a pass-through to `findBySql`, so it is not by itself enough to
make this arm meaningful.

## Converged shape

`execute(params, connection, { allowRetry = false, async = false } = {}, block?)`
branching to `asyncFindBySql` vs `findBySql` with the same argument list, and the
`RangeError` catch returning `Promise.wrap([])` on the async arm — mirroring
`statement_cache.rb:147-154` line for line.

## Acceptance criteria

- [ ] `StatementCache#execute` takes the `async` kwarg with Rails' default
      (`false`) and branches to `asyncFindBySql` / `findBySql`.
- [ ] The `RangeError` catch returns the wrapped-promise form on the async arm.
- [ ] `Association#find_target`'s statement-cache path passes `async` through,
      so the arm is actually reachable (`association.rb:267-273`).
- [ ] Both `statement-cache.json` call-set rows (`async_find_by_sql`, `wrap`) are
      deleted by hand via `serializeBaseline`, then
      `pnpm parity:api:calls:tighten activerecord/statement-cache.json`.
      No `--write`, no reseed.
- [ ] `pnpm parity:api:calls` and `pnpm parity:api:calls:args` green.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.
