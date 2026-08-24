---
title: "handle_warnings' _handleWarnings hook and _warningCount seam have no Rails counterpart"
status: closed
updated: 2026-08-24
rfc: "0119-connection-adapter-fidelity"
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
closed-reason: "Already converged on main (152b2ebe9): `_handleWarnings` and `_warningCount` do not exist anywhere under packages/*/src. Both invented seams were removed."
---

## Context

Surfaced while converging `handleWarnings`' arity in PR #6336
(`mysql2-handle-warnings-drops-conn-parameter`). That PR fixed the parameter
list; two invented seams underneath it are still there.

Rails has exactly ONE method here — `AbstractMysqlAdapter#handle_warnings(sql)`
(`vendor/rails/activerecord/lib/active_record/connection_adapters/abstract_mysql_adapter.rb:770-784`)
— carrying the whole body, and `Mysql2Adapter` does not override it. It reads
the warning count off the connection directly (`@raw_connection.warning_count`,
`:771` and `:773`).

trails splits that into three:

1. `packages/activerecord/src/connection-adapters/abstract-mysql-adapter.ts:1580`
   `handleWarnings(sql)` delegates to a `_handleWarnings(sql)` no-op hook
   (`:1585`), which `Mysql2Adapter` overrides. Rails has no such hook; the base
   class IS the implementation. The trilogy adapter, the other subclass the
   hook was shaped for, is Ruby-only and never ported.
2. `packages/activerecord/src/connection-adapters/mysql2-adapter.ts` overrides
   `handleWarnings` with the full body, so the Rails body lives on the subclass
   rather than on `AbstractMysqlAdapter` where Rails puts it.
3. `_warningCount(conn)` (same file) stands in for
   `@raw_connection.warning_count`, falling back to a `SHOW COUNT(*) WARNINGS`
   round-trip when the npm driver does not expose the protocol field. The
   fallback is a genuine driver gap, but it is reachable as a private detail of
   the ported body rather than as its own protected seam — it is only protected
   so `warnings.test.ts:111-113` can `vi.spyOn` it.

## Converged shape

One `handleWarnings(sql)` on `AbstractMysqlAdapter`, carrying `:770-784`'s body
in Rails' branch order, reading the adapter's own raw connection. No
`_handleWarnings` hook and no `Mysql2Adapter` override. Keep the driver-gap
warning-count fallback inline (or as an `@noRailsEquivalent` private with the
gap cited), and re-point the `warning_count does not match returned warnings`
test at whatever seam survives.

## Acceptance criteria

- [ ] `_handleWarnings` is gone from `abstract-mysql-adapter.ts`.
- [ ] The `handle_warnings` body lives on `AbstractMysqlAdapter`, in Rails'
      branch order, with no `Mysql2Adapter` override.
- [ ] `packages/activerecord/src/adapters/abstract-mysql-adapter/warnings.test.ts`
      stays green on both MySQL lanes, including the warning-count-mismatch test.
