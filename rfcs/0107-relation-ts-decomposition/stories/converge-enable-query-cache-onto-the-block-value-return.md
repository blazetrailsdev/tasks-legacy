---
title: "enable_query_cache/cache still adopt the block value where disable_query_cache no longer does"
status: ready
updated: 2026-08-23
rfc: "0107-relation-ts-decomposition"
cluster: null
packages: []
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

Ruby's query-cache block helpers return the block's value and run their
restore in an `ensure`, which fires when the block RETURNS — so a block
returning a pending handle (rather than rows) restores synchronously and the
handle passes through untouched:

- `QueryCache::ClassMethods#cache` / `#uncached` (activerecord/lib/active_record/query_cache.rb)
- `ConnectionPool#enable_query_cache` / `#disable_query_cache`
  (activerecord/lib/active_record/connection_adapters/abstract/connection_pool.rb)

PR #6905 converged the `uncached` / `disable_query_cache` half of that pair,
because `skip_query_cache_if_necessary` (relation.rb:1466-1471) sits on
`exec_main_query`'s path and an `async` wrapper there adopted the pending
`FutureResult` and resolved the scheduled query away.

The `cache` / `enable_query_cache` half was left `async` and still adopts:

- `packages/activerecord/src/connection-adapters/abstract/query-cache.ts` —
  `enableQueryCache` is `async` with `return await fn()`.
- `packages/activerecord/src/connection-adapters/abstract/connection-pool.ts` —
  `enableQueryCache` declares `Promise<T>`.
- `packages/activerecord/src/query-cache.ts` — the `cache` class method.

That is now an asymmetry between two halves of one Ruby pair, and any future
caller that runs a scheduled query inside `Model.cache { ... }` hits the same
adopt-the-handle bug #6905 fixed on the other side.

## Converged shape

Give `enableQueryCache` / `cache` the same non-adopting shape the `disable`
half now has: run the block, restore synchronously when the value is not a
`Promise`, and defer the restore via `.finally()` when it is. Return
`T | Promise<T>` rather than `Promise<T>`.

## Acceptance criteria

- `enableQueryCache` / `cache` are not `async` and hand the block's value
  back untouched.
- A `FutureResult` returned from inside `Model.cache { ... }` reaches the
  caller pending, not resolved.
- Existing query-cache tests green on all three lanes.
- `pnpm parity:api:calls` / `:args` add zero rows.
