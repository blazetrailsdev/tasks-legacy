---
title: "Residual build_configs / configs_for body divergences in database_configurations.rb"
status: draft
updated: 2026-08-18
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 110
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

# Residual build_configs / configs_for body divergences in database_configurations.rb

## Context

Surfaced while converging the `database-configurations.ts` call-set rows in
PR #6723 (`wave-4c-ar-core-residue-config`). That PR replaced the invented
`_buildConfigs` / `_buildConfig` pair with Rails' `build_configs` →
`walk_configs` / `build_db_config_from_raw_config` chain and routed `resolve`
and `configs_for` through their Rails callees. Three divergences inside those
bodies were left in place, each deliberately, because converging them changes
behaviour beyond that PR's scope.

**1. `configs_for` drops Rails' env default when `name:` is given.**

`activerecord/lib/active_record/database_configurations.rb:98-100`

```ruby
def configs_for(env_name: nil, name: nil, config_key: nil, include_hidden: false)
  env_name ||= default_env if name
  configs = env_with_configs(env_name)
```

trails calls `this.envWithConfigs(options.envName)` with no such default, so a
name-only lookup scans every environment instead of the default one and can
return a config from the wrong env.

**2. `build_configs` drops the `DatabaseConfigurations` arm.**

`activerecord/lib/active_record/database_configurations.rb:201-202`

```ruby
return configs.configurations if configs.is_a?(DatabaseConfigurations)
return configs if configs.is_a?(Array)
```

trails ports only the Array arm; its parameter type is
`RawConfigurations | DatabaseConfig[]`, so re-wrapping an existing
`DatabaseConfigurations` (which Rails supports) is not expressible.

**3. The three-tier test is narrower than Rails'.**

`activerecord/lib/active_record/database_configurations.rb:205`

```ruby
if config.is_a?(Hash) && config.values.all?(Hash)
```

trails' `_isThreeLevelConfig` additionally rejects a hash carrying
`adapter`/`url`/`database`, and rejects the empty hash. Rails treats `{}` as
three-tier (`walk_configs` over no entries yields `[]`), where trails builds a
`primary` HashConfig for it — so `{ development: {} }` produces one config in
trails and none in Rails.

Explicitly **not** in scope, already tracked:
`converge-databaseconfigurations-build-entry-points-onto-new` (the
constructor/`fromRaw`/`fromEnv` split and the `currentEnv` build env) and
`converge-configs-for-name-returns-single-config` (the `name:` return shape).

## Converged shape

- `configsFor` opens with Rails' `env_name ||= default_env if name`.
- `buildConfigs` accepts and unwraps a `DatabaseConfigurations`, widening the
  parameter type to match `:201`.
- The three-tier test becomes `config.values.all?(Hash)` — or, if the
  `adapter`/`url`/`database` rejection is genuinely load-bearing because TS
  object configs cannot be told apart the way symbol-keyed Ruby hashes can,
  that narrowing is justified at the call site with `:205` and the empty-hash
  arm is still fixed to match Rails.

## Acceptance criteria

- [ ] `configsFor` applies the default env when `name` is given and `envName`
      is not, with a test covering a name-only lookup that would otherwise
      match a config in a non-default env; it must fail on baseline.
- [ ] `buildConfigs` handles a `DatabaseConfigurations` argument per `:201`.
- [ ] `{ development: {} }` yields the same config count as Rails.
- [ ] `pnpm parity:api:calls` / `:args` clean, no new baseline rows.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.
