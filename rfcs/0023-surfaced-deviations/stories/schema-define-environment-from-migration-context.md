---
title: "Schema#define stamps a bespoke environment chain instead of migration_context.current_environment"
status: draft
updated: 2026-08-24
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
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

Rails' `Schema::Definition#define` stamps the environment with
`connection_pool.internal_metadata.create_table_and_set_flags(connection_pool.migration_context.current_environment)`
(`activerecord/lib/active_record/schema.rb:54-62`) — the label comes from the
pool's `migration_context`.

`packages/activerecord/src/schema.ts` `define` (reshaped in PR #7008 to Rails'
`new.define(info, &block)` + `connection_pool.with_connection` form) resolves it
through a trails-only chain instead:
`info.environment ?? getEnv("TRAILS_ENV") ?? getEnv("NODE_ENV") ??
DatabaseConfigurations.defaultEnv`. The `info.environment` key is not part of
Rails' `info` hash at all (Rails' `info` carries only `:version`), and the
`NODE_ENV` arm is a one-release fallback.

This is the last shape difference left in that method after #7008; the
`with_connection` lease, the yielded connection, `assume_migrated_upto_version`
and `info[:version].present?` all match now.

## Converged shape

`define` reads `this.connectionPool.migrationContext.currentEnvironment` the way
Rails does, and `SchemaDefineInfo` loses the `environment` key (Rails' `info` has
only `:version`). Callers that need a different environment set it where the
migration context reads it, not through a bespoke `info` key.

## Acceptance criteria

- [ ] `Schema#define` passes `connectionPool.migrationContext.currentEnvironment`
      to `createTableAndSetFlags`, citing `schema.rb:61`.
- [ ] `SchemaDefineInfo.environment` is gone, or carries a `@noRailsEquivalent`
      with a reviewed reason if a caller genuinely cannot be moved.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.
