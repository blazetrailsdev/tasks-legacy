---
title: "DatabaseTasks.env and currentEnv() are two resolvers where Rails has one"
status: ready
updated: 2026-08-22
rfc: "0112-one-rails-thing-n-trails-things"
cluster: split-stores
deps: []
deps-rfc: []
est-loc: 90
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Rails' `db:*` tasks resolve one environment: `DatabaseTasks.env` is
`ActiveRecord::ConnectionHandling::DEFAULT_ENV.call` (`tasks/database_tasks.rb`),
which is `Rails.env` — the same value every task and the boot connection see.

trails has two resolvers that can disagree:

- `DatabaseTasks.env` (`packages/activerecord/src/tasks/database-tasks.ts:52-54`)
  returns `DatabaseConfigurations.defaultEnv`, which falls back to the literal
  `"development"`.
- `DatabaseConfigurations.currentEnv()` (`packages/activerecord/src/database-configurations.ts:134-141`)
  resolves `TRAILS_ENV` → `_defaultEnv` → `NODE_ENV` → default.

`DatabaseConfigurations.fromEnv` builds configs under `currentEnv()` but never
assigns `defaultEnv`, so with `TRAILS_ENV=test` and no explicit assignment the
two disagree. In `activerecord-cli` this forced per-call-site choices (PR #5735):
`dbMigrate` / `dbRollback` must pass `DatabaseTasks.env` because `migrateAll` and
`rollback` resolve through `_normalizeEnv()` → `DatabaseTasks.env`
(`database-tasks.ts:619-622, 1048, 406`), while `console.ts` / `runner.ts` and the
rest of `db-tasks.ts` use `currentEnv()`. A default on
`withEnvironmentConnection` was deliberately removed for this reason: it would
silently boot one env while the task migrated another.

Related drafts: `current-env-trails-env-outranks-default-env` (precedence inside
`currentEnv`), `default-env-literal-is-not-default-env`. This story is about the
two resolvers coexisting, not the ordering within either.

## Acceptance criteria

- [ ] One env resolution reaches both the task layer and the config lookup, as
      Rails' single `Rails.env` does.
- [ ] `activerecord-cli` call sites no longer have to choose between
      `DatabaseTasks.env` and `DatabaseConfigurations.currentEnv()`.
- [ ] A regression test covers `TRAILS_ENV` set with `defaultEnv` unassigned —
      the case where the two currently disagree.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
