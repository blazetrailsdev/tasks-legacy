---
title: "checkout/leaseConnection do not await pool.adapterReady, forcing manual awaits at call sites"
status: draft
updated: 2026-07-28
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
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

`ConnectionPool#newConnection` (connection-adapters/abstract/connection-pool.ts:1285)
delegates to `DatabaseConfig#newConnection`
(database-configurations/database-config.ts:179), which resolves the adapter
class **synchronously** and throws
`Adapter "<x>" not pre-resolved — await pool.adapterReady or this.loadAdapter()`
(database-config.ts:189-197) when the class has not finished loading.

`ConnectionHandler#establishConnection` (connection-handler.ts:178-182) only
kicks off `resolveConnectionAdapter` and stashes the promise on
`pool.adapterReady`; neither `leaseConnection` (connection-pool.ts:648) nor
`checkout` awaits it. Every caller that leases soon after establishing must
therefore `await pool.adapterReady` by hand, or race the dynamic import — a
latent order-dependent failure that only hides because the adapter module
cache is usually already warm from an earlier test in the same file.

Surfaced on PR #5494, where three tests in `connection-handling.test.ts`
(`AbstractAdapter#isPreventingWrites stack matching`) had to add explicit
`await pool.adapterReady` calls after dropping their injected `adapterFactory`.

Rails has no analogue: `db_config.new_connection` resolves the adapter class
through a plain `require`, so there is nothing to await and no
`adapter_ready` on `ConnectionPool`. The manual-await requirement is a trails
invention leaking into call sites.

## Acceptance criteria

- [ ] `checkout` / `leaseConnection` (or `newConnection`'s call path) await the
      pending adapter load itself, so callers never need `pool.adapterReady`.
- [ ] The "not pre-resolved" throw remains reachable only for genuine loader
      failures, not for a merely-pending load.
- [ ] Existing `await pool.adapterReady` call sites become unnecessary; remove
      the ones in `connection-handling.test.ts` as proof.
- [ ] All three lanes green.
