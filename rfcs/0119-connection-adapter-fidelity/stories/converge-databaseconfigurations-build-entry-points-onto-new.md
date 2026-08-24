---
title: "Converge DatabaseConfigurations build entry points onto Rails' single constructor"
status: ready
updated: 2026-07-27
rfc: "0119-connection-adapter-fidelity"
cluster: null
deps: []
deps-rfc: []
est-loc: 90
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Rails has exactly one way to build a `DatabaseConfigurations`:
`DatabaseConfigurations.new(config)`, which calls `build_configs` and merges
`DATABASE_URL` (`vendor/rails/activerecord/lib/active_record/database_configurations.rb`).
`Base.configurations=` is spelled `@@configurations =
ActiveRecord::DatabaseConfigurations.new(config)` (core.rb:71-73).

trails has three entry points in
`packages/activerecord/src/database-configurations.ts`, differing only in which
env they resolve against and whether they register `DatabaseConfigurations.current`:

- `constructor(raw)` — `_buildConfigs(raw, DatabaseConfigurations._defaultEnv)`, registers `current`
- `static fromRaw(raw)` — `_buildConfigs(raw, DatabaseConfigurations.defaultEnv)`, registers `current`
- `static fromEnv(raw)` — `_buildConfigs(raw, DatabaseConfigurations.currentEnv())`, does register `current` (via the `new ... ([])` it delegates to)

PR #5381 wired `Core.configurations` (`packages/activerecord/src/core.ts`) as
the Rails-named writer. It had to pick one, and picked `fromEnv` — not the
constructor Rails uses — because every runtime config selector in
`connection-handling.ts` looks configs up by `currentEnv()`, so building under
`_defaultEnv` would resolve to configs nothing later finds. That is a
deviation held in place by the three-way split, not by anything Rails does.

## Acceptance criteria

- One build path, spelled as Rails' constructor: `new DatabaseConfigurations(raw)`
  is the only way raw config becomes `DatabaseConfig` objects.
- The env a config is built under and the env it is later looked up by are the
  same value; `currentEnv()` / `defaultEnv` / `_defaultEnv` collapse to
  whichever one Rails' `DEFAULT_ENV` actually corresponds to (see
  `connection_handling.rb`'s `DEFAULT_ENV`), with the others deleted or
  documented at the call site as to why they differ.
- `Core.configurations`'s writer calls that constructor, matching core.rb:72.
- `DatabaseConfigurations.current` registration happens in exactly one place.
- `pnpm typecheck`, `pnpm lint`, and the database-configurations,
  connection-handling, connection-handlers-multi-db,
  connection-handlers-sharding-db, test-databases and tasks/database-tasks
  suites pass.
