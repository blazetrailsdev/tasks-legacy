---
title: "Converge get_primary_key's tableExists / primaryKeys receipts once RFC 0073 lands"
status: blocked
updated: 2026-08-24
rfc: "0106-wide-call-set-direct-burndown"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 120
priority: null
pr: null
claim: "2026-08-24T18:07:08Z"
assignee: "stale-story-references-scan-times-out-under-load"
blocked-by: "Gated on RFC 0073 (permanent-connection-checkout flip), still status: draft — unlanded. getPrimaryKey / getPrimaryKeyAttr / primaryKey run on synchronous paths (model construction), while trails' ported tableExists (model-schema.ts:1470) and SchemaCache#primaryKeys are async; their sync cache-only views (cachedTableExists, getCachedPrimaryKeys) exist precisely because no lease-free synchronous read is expressible until 0073 settles the lease shape. Converging now would either lease a connection from a sync getter (the deviation the receipts exist to avoid) or make primaryKey async, breaking every sync caller. Unblock when 0073 lands."
closed-reason: null
---

## Context

`getPrimaryKey` (`packages/activerecord/src/attribute-methods/primary-key.ts`)
carries two `@missingRailsCall … — CONVERGEABLE` receipts, landed by PR #6987
in place of the two `kind: "set"` baseline rows that used to track them:

- `table_exists?` — `primary_key.rb:103`
- `primary_keys` (`schema_cache.primary_keys`) — `primary_key.rb:104`

Rails:

    elsif ActiveRecord::Base != self && table_exists?   # primary_key.rb:103
      schema_cache.primary_keys(table_name)             # primary_key.rb:104

trails reaches both values through a lease-free cache-only path
(`cachedSchemaCacheFor(this)?.getCachedPrimaryKeys?.(tableName)`) because
`tableExists` and `schemaCache.primaryKeys` are async here, and the sync
cache-only view of `tableExists` (`cachedTableExists`) leases a connection —
which `getPrimaryKey` must not do: `getPrimaryKeyAttr` / `primaryKey` are read
from synchronous paths (model construction).

The receipts are debt, not permission — they are classified CONVERGEABLE
precisely because a synchronous schema-cache read IS expressible in TypeScript
once RFC 0073 settles the lease shape.

## Converged shape

Once `withConnection` / the permanent-checkout flip lands, `getPrimaryKey` calls
the ported `tableExists` and `primaryKeys` names directly, in Rails' branch
order, and both `@missingRailsCall` tags are deleted from the JSDoc. No baseline
row is re-added.

## Acceptance criteria

- [ ] `getPrimaryKey` issues the ported `tableExists` / `primaryKeys` calls,
      matching `primary_key.rb:101-108`.
- [ ] Both `@missingRailsCall` tags are gone from
      `packages/activerecord/src/attribute-methods/primary-key.ts`.
- [ ] No connection is leased from `getPrimaryKey` / `getPrimaryKeyAttr` — the
      synchronous callers still work with a cold cache and no configured
      connection.
- [ ] `pnpm parity:api:calls` / `:args` green with no new baseline rows.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.
