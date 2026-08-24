---
title: "validateBang defers a registered adapter's load failure to first checkout"
status: draft
updated: 2026-07-28
rfc: "0119-connection-adapter-fidelity"
cluster: null
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

Rails' `DatabaseConfig#validate!`
(`vendor/rails/activerecord/lib/active_record/database_configurations/database_config.rb:29-33`)
calls `adapter_class`, which in
`vendor/rails/activerecord/lib/active_record/connection_adapters.rb:41-52`
both looks the name up in the registry AND `require`s the adapter file,
rescuing `LoadError` into a "Error loading the '<name>' Active Record adapter"
`LoadError`. So Rails surfaces a _broken but registered_ adapter at
`establish_connection` time.

PR #5515 ported only the first half. `DatabaseConfig#validateBang`
(`packages/activerecord/src/database-configurations/database-config.ts`) is
sync and calls `validateAdapterName`
(`packages/activerecord/src/connection-adapters.ts`), a registry-membership
check. That was deliberate: `adapterClass()` is async (ESM imports are) while
`resolvePoolConfig` and `establishConnection` are sync, and floating a promise
was ruled out.

Consequence: `register("x", () => import("broken"))` followed by
`establishConnection({ adapter: "x" })` succeeds; the import failure only
surfaces at first connection checkout, via
`newConnection`'s `Adapter "x" not pre-resolved — loader failed: ...` message,
which is a trails invention with no Rails counterpart.

## Acceptance criteria

- [ ] Decide the bridge: either make the `establishConnection` path async (and
      audit its many sync callers), or pre-resolve the adapter eagerly enough
      that `validateBang` can report a load failure synchronously.
- [ ] A registered-but-unloadable adapter fails at `establishConnection`, not
      at first checkout.
- [ ] The raised error matches Rails' shape from `connection_adapters.rb:44-51`
      ("Error loading the '<name>' Active Record adapter. Missing a gem it
      depends on? <message>"), adapted for the JS package/loader equivalent.
- [ ] No floated promises.
