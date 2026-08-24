---
title: "converge-sync-connection-lease-per-checkout-verify"
status: draft
updated: 2026-08-03
rfc: "0119-connection-adapter-fidelity"
cluster: null
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`connection-pool-pinned-sync-checkout-per-checkout-verify` landed as #4443, but
the residual it describes is still in the tree at three sites, each citing the
landed story as its tracker:

- `packages/activerecord/src/connection-adapters/abstract/connection-pool.ts:636-660`
  — `leaseConnectionSync()`, the sync lease that skips the async per-checkout
  `verifyBang`. It is `@internal` and already carries
  `@noRailsEquivalent CONVERGEABLE`.
- `packages/activerecord/src/connection-handling.ts:499-509` — the deprecated
  sync `.connection` getter routes through `leaseConnectionSync`, losing the
  self-heal Rails' `lease_connection` performs there.
- `packages/arel/src/nodes/node.ts:49-60` — `toSql` takes the sync
  `engine.connection` lease in place of Rails'
  `engine.with_connection { |c| ... }` (`arel/nodes/node.rb:148-153`), skipping
  the same verify.

The root cause is unchanged: trails' Rails-named `leaseConnection` / `checkout`
became async (they await per-checkout `verifyBang`), while all three call sites
are synchronous — `to_sql` alone is sync at 600+ call sites. The
`@noRailsEquivalent` tag at connection-pool.ts:655 names
`retire-connection-pool-async-resolution-shims` as the convergence owner, so
this residual should either fold into that story or be tracked here.

See also `project_sync_active_getter_drops_rails_live_probe` and
`project_pool_adapter_proxy_makes_sync_methods_async`.

## Acceptance criteria

- The per-checkout `verifyBang` self-heal is restored on the sync paths, or
  `leaseConnectionSync` is retired in favour of the async Rails-named surface
  at all three sites.
- All three comments stop citing the landed
  `connection-pool-pinned-sync-checkout-per-checkout-verify` and either drop
  the deviation note or cite the story that actually owns it.
- `scripts/stale-story-references.test.ts` stays green.
