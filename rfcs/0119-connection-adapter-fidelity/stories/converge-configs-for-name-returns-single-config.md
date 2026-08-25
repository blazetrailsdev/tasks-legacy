---
title: "configs_for(name:) returns a single DatabaseConfig, not an array"
status: ready
updated: 2026-08-25
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
closed-reason: null
---

## Context

Surfaced while converging `DatabaseTasks.configs_for` in PR #6360
(`call-args-ar-kwargs-vs-positional`). The call sites now pass Rails' kwargs,
but the callee's return shape still diverges.

Rails `DatabaseConfigurations#configs_for` returns a **single**
`DatabaseConfig` (not an array) when `name:` is given, and `nil` when nothing
matches — see `activerecord/lib/active_record/database_configurations.rb`
(`configs_for(env_name: nil, name: nil, config_key: nil, include_hidden: false)`;
the `name` arm ends in `.find { ... }`). That is what lets
`database_tasks.rb:327-340` write:

```ruby
def charset_current(env_name = env, db_name = name)
  db_config = configs_for(env_name: env_name, name: db_name)
  charset(db_config)
end
```

— passing `db_config` straight to `charset`, which expects one config.

trails' `DatabaseConfigurations#configsFor`
(`packages/activerecord/src/database-configurations.ts:224`) always returns
`DatabaseConfig[]`, so `charsetCurrent` / `collationCurrent` have to index
`dbConfig[0]` and guard on `.length === 0`. Same at
`database_tasks.rb:328,338,514` and `tasks/database-tasks.ts:1344-1349`.

## Converged shape

`configsFor` returns a single `DatabaseConfig | undefined` when `name` is
present and `DatabaseConfig[]` otherwise (a TS overload pair), so the callers
drop their array indexing and `.length` guards.

## Acceptance criteria

1. `configsFor` matches Rails' return shape on both arms, cited to
   `database_configurations.rb`.
2. `charset_current`, `collation_current`, and the `configs_for(env_name:,
name:)` sites in `tasks/database-tasks.ts` pass the result straight through,
   as `database_tasks.rb:327-340` does.
3. Every other `configsFor` caller across `activerecord`, `activerecord-cli`,
   and `trailties` is updated.
4. `pnpm parity:api` and `pnpm parity:test` deltas non-negative.

Note: `converge-charset-current-collation-current-onto-configs-for-pair`
(0023, draft) covers the call-site half and was largely satisfied by PR #6360 —
check whether it is now stale before starting.
