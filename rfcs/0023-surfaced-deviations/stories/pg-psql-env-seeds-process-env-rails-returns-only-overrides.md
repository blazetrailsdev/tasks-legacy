---
title: "psqlEnv seeds process.env where Rails returns only the PG* overrides and lets Kernel.system merge"
status: draft
updated: 2026-08-05
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
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

Rails' `PostgreSQLDatabaseTasks#psql_env`
(`vendor/rails/activerecord/lib/active_record/tasks/postgresql_database_tasks.rb:105-114`)
builds a **fresh** hash and returns only the PG\* variables it sets:

    def psql_env
      {}.tap do |env|
        env["PGHOST"] = db_config.host if db_config.host
        ...
      end
    end

`Kernel.system(psql_env, cmd, *args)` (`:117`) merges that over the inherited
environment, so the child gets the parent env plus the overrides.

trails' `psqlEnv`
(`packages/activerecord/src/tasks/postgresql-database-tasks.ts:243-258`) instead
seeds the hash with a full copy of `process.env` and passes the result as
`spawnSync`'s `env`, because `spawnSync` **replaces** the child environment
rather than merging. The guard chain itself was converged in #6141 and matches
Rails line-for-line; the seeding is the remaining deviation.

Observable difference: any PG\* variable already in the parent environment
(`PGPASSWORD`, `PGSSLMODE`, `PGSERVICE`, …) leaks into the child even when the
db_config sets none, where Rails would still pass it through via the shell's own
inheritance — so in the common case the two coincide, but trails also copies
every unrelated variable into an explicit env, which changes what a
`spawnSync` mock observes and makes the method's return value not the
"just the overrides" hash Rails documents.

Surfaced while converging `psqlEnv`'s readers in #6141
(`pg-database-tasks-reads-db-config-not-a-hand-parsed-url`); scoped out because
it is a process-spawning change, not a config-reader change.

## Converged shape

`psqlEnv` returns only the PG\* overrides, as `postgresql_database_tasks.rb:105-114`
does. `runCmd` does the merging at the spawn boundary — the `{ ...parentEnv,
...this.psqlEnv() }` spread moves into `runCmd`, which is where Ruby's
`Kernel.system` merge semantics actually live.

## Acceptance criteria

- [ ] `psqlEnv` starts from an empty object and contains only the eight PG\*
      assignments of `postgresql_database_tasks.rb:106-113`.
- [ ] `runCmd` performs the parent-env merge, mirroring `Kernel.system(env, ...)`
      at `:117`.
- [ ] A test asserts `psqlEnv()` returns ONLY the configured PG\* keys.
- [ ] PG lane green.
