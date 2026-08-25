---
title: "PG: port is_cached_plan_failure? / PreparedStatementCacheExpired retry"
status: ready
updated: 2026-07-27
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 150
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Rails' PG adapter classifies stale-prepared-plan errors via
`is_cached_plan_failure?` (postgresql_adapter.rb:890-906: SQLSTATE 0A000 +
source function `RevalidateCachedQuery`) and raises
`PreparedStatementCacheExpired`, which the transaction machinery uses to
deallocate and retry outside a transaction. trails has none of this: PR #5094
worked around the common DDL-driven case by resetting the statement-name map in
`reloadTypeMap` (postgresql-adapter.ts) after type DDL, but any OTHER
server-side plan invalidation (e.g. `ALTER TABLE` changing a SELECT's result
type, search_path changes) still surfaces the raw PG error
("cached plan must not change result type" / "cache lookup failed for type")
instead of Rails' retry-or-PreparedStatementCacheExpired behavior. Port the
classification + retry: `is_cached_plan_failure?`, `in_transaction?` gate, and
the exec_cache rescue path (postgresql_adapter.rb around the adapter's
`exec_cache`/`with_raw_connection` retry).

## Acceptance criteria

- [ ] PG adapter classifies cached-plan failures like Rails
      (`is_cached_plan_failure?`) and raises PreparedStatementCacheExpired
      inside transactions.
- [ ] Outside a transaction, the statement is deallocated and retried once.
- [ ] parity:test / parity:api delta non-negative.
