---
title: "AbstractSQLite3Adapter connects in its constructor; Rails leaves raw_connection nil until connect!"
status: ready
updated: 2026-07-27
rfc: "0119-connection-adapter-fidelity"
cluster: null
deps: []
deps-rfc: []
est-loc: 80
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced on PR #5301 (`lease-connection-leaves-raw-connection-unestablished`).

Rails' `AbstractAdapter#initialize` sets `@raw_connection = nil`
(vendor/rails/activerecord/lib/active_record/connection_adapters/abstract_adapter.rb:128),
so `connected?` — `!@raw_connection.nil?` (abstract_adapter.rb:649) — is
**false** for a freshly constructed/checked-out adapter on every adapter,
sqlite3 included. Rails' `test_new_connection_no_query`
(connection_pool_test.rb:105) and `test_connected?`
(connection_handlers_multi_pool_config_test.rb:73, which must call
`pool.checkout.connect!` to make `connected?` true) both depend on that.

trails' `AbstractSQLite3Adapter` constructor instead calls `this.connect()`
(packages/activerecord/src/connection-adapters/sqlite3-adapter.ts:395), opening
the raw driver handle eagerly. Verified with a probe on a fresh pool built from
`newRawTestAdapter` (not the ambient pool, so no prior query can explain it):

```text
sqlite lane: isConnected after checkout: true   poolIsConnected: true
pg lane:     isConnected after checkout: false  poolIsConnected: false
```

The comment at the call site justifies it as leak-avoidance for the native
driver's file handle, and async-only drivers (expo-sqlite) are already flagged
onto an async path by `connect()` — so a lazy path partly exists.

**Why this matters beyond tidiness:** the eager connect makes the sqlite lane
report `connected? == true` where Rails reports false, which _masks_
lazy-connect divergences. PR #5301's `is_connected?` case and the
`checkout`/`leaseConnection` stories (#5159, #5301) were all diagnosed as
PG/MySQL-only failures purely because sqlite silently passed. Any future
assertion about pre-connect state is untrustworthy on the default lane.

## Acceptance criteria

- [ ] A freshly constructed `AbstractSQLite3Adapter` has no open raw handle:
      `isConnected()` is false until `verifyBang()` / first query, matching
      Rails and the PG/MySQL lanes.
- [ ] The file-handle leak the eager `connect()` guards against is handled on
      the lazy path (no leaked handles across pool disconnect/reap suites).
- [ ] `connection-pool.test.ts` "new connection no query" still passes on the
      sqlite lane, and `connection-handling.test.ts` `is_connected?` still
      passes on all three lanes.
- [ ] expo-sqlite / libsql async-driver paths unaffected.
