---
title: "DatabaseTasks target-version options stand in for ENV[VERSION]"
status: draft
updated: 2026-08-02
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
  - "activesupport"
  - "trailties"
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

Rails' `db:migrate` passes no argument to `DatabaseTasks.migrate` /
`migrate_all`; the target version arrives as `ENV["VERSION"]`, read by
`DatabaseTasks.target_version`
(`activerecord/lib/active_record/tasks/database_tasks.rb`, `railties/databases.rake:88`).

PR #5864 could not reproduce that from the CLI. `DatabaseTasks.targetVersion()`
reads `getEnv` (`packages/activesupport/src/environment.ts`), which is a
read-only wrapper over `process.env`, and trailties may not touch `process.*`.
So `migrate`, `migrateAll` and `dbConfigsWithVersions` each gained a
`targetVersion` option that stands in for the env read
(`packages/activerecord/src/tasks/database-tasks.ts`), and
`packages/trailties/src/commands/db.ts` passes `--version` through it.

Note there are two env surfaces: `@blazetrails/activesupport/process-adapter`
exports a writable `setEnv` (used by `db.test.ts`), while `DatabaseTasks` reads
the separate `activesupport/src/environment.ts` `getEnv`. Converging those two
would let the CLI set `VERSION` the way Rails does and delete all three
option parameters.

## Acceptance criteria

- [ ] `DatabaseTasks.targetVersion()` and `checkTargetVersion` read an env
      surface the CLI can write, or the deviation is justified in place.
- [ ] The `targetVersion` option on `migrate`, `migrateAll` and
      `dbConfigsWithVersions` is removed once the env path works.
- [ ] `db migrate --version` and `VERSION=x trails db migrate` keep the
      "migrate up to this version" semantics pinned by
      "db migrate --version migrates up to that version, not only that version"
      in `packages/trailties/src/commands/db.test.ts`.
