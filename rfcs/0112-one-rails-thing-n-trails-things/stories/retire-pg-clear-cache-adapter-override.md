---
title: "Retire the PostgreSQL clear_cache! override — Rails has no adapter override"
status: ready
updated: 2026-08-25
rfc: "0112-one-rails-thing-n-trails-things"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 90
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Rails has **no adapter override of `clear_cache!`**. The whole body lives once
on the base:

```ruby
# activerecord/lib/active_record/connection_adapters/abstract_adapter.rb:739-748
def clear_cache!(new_connection: false)
  if @statements
    @lock.synchronize do
      if new_connection
        @statements.reset
      else
        @statements.clear
      end
    end
  end
end
```

PR #7050 (RFC 0112) converged mysql2 and sqlite3 onto that inherited body and
deleted their overrides. **PostgreSQL still overrides it**
(`packages/activerecord/src/connection-adapters/postgresql-adapter.ts`,
`clearCacheBang`), for two pieces of trails-only state:

- `_rawConnection` — when the handle is torn down, the override resets the pool
  instead of clearing it, because a `DEALLOCATE` on a dead session would throw.
- `_needsDeallocateAll` — a flag the next acquire drains, standing in for the
  server-side statements that never got deallocated.

Rails needs neither, because its `PostgreSQL::StatementPool` holds the
**adapter** and re-reads the raw handle at dealloc time
(`postgresql_adapter.rb:294-316`), so a reconnect invalidates the whole pool and
there is no torn-down-handle case to special-case in `clear_cache!`.

## Why it matters beyond tidiness

The override is a live footgun. It shadows the base method, so
`AbstractAdapter.prototype.clearCacheBang` is _not_ the function PG dispatches
to. PR #7050 had to rewrite `transactions.trails.test.ts`'s spy
(`clearCacheBangOwner`, which walks the prototype chain to the owner) after
removing the override's redundant `super.clearCacheBang(...)` call — that super
call was the only reason a prototype spy ever fired on PG. Any future test that
spies the prototype will silently no-op on the PG lane.

## Converged shape

Retire the PG override entirely; PG inherits `abstract_adapter.rb:739-748` like
every other adapter. That requires the `_rawConnection` / `_needsDeallocateAll`
arms to move into the pool's `dealloc`, which is where Rails puts the
equivalent concern.

**Depends on `pg-statement-pool-holds-the-connection`**
(`0085-pg-cancel-query-rails-convergence`, draft): once the pool holds the
adapter rather than a pinned `pg.Client`, it can decide at dealloc time whether
the handle is live, and the override's reason to exist goes away. Do that story
first; this one is its payoff.

Once the override is gone, `clearCacheBangOwner` in
`packages/activerecord/src/transactions.trails.test.ts` collapses back to a
plain `AbstractAdapter.prototype` spy — delete the helper with it.

## Acceptance criteria

- [ ] `postgresql-adapter.ts` has no `clearCacheBang` override; all three
      adapters use the inherited body.
- [ ] `_needsDeallocateAll`'s behaviour is preserved (or shown unnecessary),
      with the decision made inside the pool's `dealloc`.
- [ ] `clearCacheBangOwner` is deleted from `transactions.trails.test.ts` and
      both call sites spy `AbstractAdapter.prototype` directly.
- [ ] PG lane green, including the `QueryCanceled` regression #6606 fixed.
