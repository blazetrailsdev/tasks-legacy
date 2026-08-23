---
title: "reset must cancel the parked future result, not just drop it"
status: done
updated: 2026-08-23
rfc: "0107-relation-ts-decomposition"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 220
priority: 3
pr: 6905
claim: "2026-08-23T10:57:34Z"
assignee: "converge-relation-loaded-arm-readers-onto-seams"
blocked-by: null
closed-reason: null
---

# `reset` must cancel the parked future result, not just drop it

## Context

`split-future-result-scheduled-dispatch-out-of-exec-queries` (PR #6750) gave
trails Rails' `@future_result` dispatch: `loadAsync` parks the scheduled query's
handle in `_futureResult`, `isScheduled` reads it, and `execQueries` drains it
through the `if scheduled?` arm.

Rails' `reset` does two things to that handle
(`vendor/rails/activerecord/lib/active_record/relation.rb:1195-1196`):

```ruby
@future_result&.cancel
@future_result = nil
```

trails' `reset` (`packages/activerecord/src/relation.ts`, the `reset()` body)
only does the second. The `cancel` half is dropped, and a `_loadToken` bump
stands in so an in-flight load cannot re-populate `_records` after the reset.

That stand-in covers the _staleness_ hazard but not the _work_ hazard: the
query still runs to completion against the pool after the relation was reset,
where Rails stops it.

## Why the cancel is currently unreachable

trails HAS a real `FutureResult#cancel` (`packages/activerecord/src/future-result.ts`,
mirroring `future_result.rb:88-92`), and `selectAll(..., { async: true })`
genuinely returns a `FutureResult` / `Complete`. The port cannot reach it
because `execMainQuery` is an `async` method: a JS `async` function awaits any
thenable it returns, and `FutureResult` is a thenable (it defines `then`, see
`future-result.ts`). So the `FutureResult` is resolved into rows at the
`return` boundary and what `loadAsync` can park is a `Promise<Result>`, which
has no `cancel`.

This was documented at the field in PR #6750 rather than converged, because
un-`async`-ing `execMainQuery` is not a local edit — its body awaits
`skipQueryCacheIfNecessary` and the eager `applyJoinDependency` arm.

## Converged shape

- Let `execMainQuery` hand back the scheduled `FutureResult` unresolved, so
  `_futureResult` holds what `relation.rb:1148` holds. This is the whole
  problem: it means `execMainQuery`'s async arm must not sit behind an `await`
  boundary that adopts the thenable.
- `reset()` then calls `_futureResult?.cancel()` before clearing, mirroring
  `relation.rb:1195-1196`.
- `execQueries`' scheduled arm calls `.result()` on it, which is literally
  `future.result` (`relation.rb:1408`) instead of awaiting a promise.
- Keep the `_loadToken` guard: it covers the separate trails-only race where a
  reset lands mid-await, which Rails cannot have because `exec_queries` is
  synchronous.

## Acceptance criteria

- [ ] `_futureResult` holds a `FutureResult` (or `Complete`), not a
      `Promise<Result>`.
- [ ] `reset()` cancels it before clearing, as `relation.rb:1195-1196` does.
- [ ] `execQueries`' scheduled arm drains it via `result()`
      (`relation.rb:1408`).
- [ ] A regression test proves a `loadAsync()` + `reset()` cancels the
      scheduled query (fails on the baseline, where the query still completes).
- [ ] `relation-load-async.trails.test.ts` and `relation/load-async.test.ts`
      green on all three lanes; `parity:api` `relation.rb` -> `relation.ts`
      stays 401/401; `parity:api:calls` / `:args` / `:extra` clean.
