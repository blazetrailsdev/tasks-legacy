---
title: "cacheSql: hit-path instrumentation and dup split out of Rails' single cache_sql"
status: draft
updated: 2026-07-30
rfc: "0119-connection-adapter-fidelity"
cluster: null
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

Rails' `cache_sql` (`vendor/rails/activerecord/lib/active_record/connection_adapters/abstract/query_cache.rb`, `def cache_sql(sql, name, binds)`) is a single method that does four things inside one `@lock.synchronize`: computes the key, calls `@query_cache.compute_if_absent` with a bare `yield` for the miss path, instruments `sql.active_record` with `cache_notification_info_result` on the _hit_ path, and returns `result.dup`.

trails splits this across two functions in `packages/activerecord/src/connection-adapters/abstract/query-cache.ts`:

- `lookupSqlCache(sql, name, binds)` — synchronous, does the hit lookup **and** owns the hit-path `Notifications.instrument` call.
- `cacheSql(sql, name, binds, block)` — async, only handles the miss path (`qc.computeIfAbsent(key, block)`), plus a `if (!qc) return block()` guard Rails has no analogue for. It emits no instrumentation and does no `.dup` (the copy happens inside `Store.computeIfAbsent`'s row spread).

The split is deliberate and has an in-file rationale (`query-cache.ts` ~line 496-514): the caller must not `await` between the lookup and the miss-execute, so the hit path was pulled into a sync function. It is nonetheless a shape deviation from Rails, and neither the `@lock.synchronize` nor `result.dup` has a direct counterpart.

Surfaced during PR #5633 (story `arity-cache-sql-ported-block-param`), which only renamed the trailing block param and did not touch the split.

## Acceptance criteria

- Decide whether the two-function split is required by the async port or whether `cacheSql` can absorb the hit path + instrumentation and return a copy, matching Rails' single-method shape.
- If it converges: `cacheSql` owns the hit instrumentation and the result copy; `lookupSqlCache` either disappears or becomes a pure lookup with no notification side effect. Callers in `query-cache.ts` updated; `query-cache.test.ts` and `query-cache.trails.test.ts` stay green.
- If it cannot converge: keep the split, but ensure the rationale comment names the exact Rails lines it deviates from, and record the deviation wherever converged-deviation tracking lives rather than only in a code comment.
