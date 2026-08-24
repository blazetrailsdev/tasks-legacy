---
title: "migrate-status-aborts-as-error-and-invents-a-memory-database-fallback"
status: in-progress
updated: 2026-08-24
rfc: "0051-migration-schema-statements-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 50
priority: null
pr: 6980
claim: "2026-08-24T12:51:22Z"
assignee: "converge-schema-cache-install-onto-cache-replacement"
blocked-by: null
closed-reason: null
---

## Context

`DatabaseTasks.migrateStatus`
(`packages/activerecord/src/tasks/database-tasks.ts`) is the port of
`ActiveRecord::Tasks::DatabaseTasks#migrate_status`
(`vendor/rails/activerecord/lib/active_record/tasks/database_tasks.rb:302-315`).
PR #6977 kept it — the RFC 0051 story
`database-tasks-rake-only-migration-methods-move-to-callers` had listed it for
deletion on the premise that Rails has no counterpart, which is wrong:
`databases.rake:230-232` _calls_ it rather than inlining it, unlike the four
methods that story did retire. But the body still diverges in two ways:

1. **Missing-table arm.** Rails is
   `Kernel.abort "Schema migrations table does not exist yet."` (`:304`); trails
   throws `new Error(...)` with the same string. `Kernel.abort` exits the
   process — the CLI's exit-code path, not an exception a caller can catch.
2. **Invented database fallback.** Rails prints
   `"\ndatabase: #{migration_connection_pool.db_config.database}\n\n"` (`:308`);
   trails prints `pool.dbConfig.database ?? ":memory:"`. The `":memory:"`
   substitute has no Rails counterpart and changes the output for any config
   whose `database` is nil.

Also worth checking while in there: Rails re-reads `migration_connection_pool`
at each of the three use sites (`:303`, `:308`, `:311`); trails hoists it into a
local `pool`.

## Converged shape

The missing-table arm goes through whatever trails uses for `Kernel.abort`'s
process-exit semantics rather than a catchable `Error`, and the header prints
`db_config.database` bare, with no `":memory:"` substitute.

## Acceptance criteria

- [ ] The `":memory:"` fallback is gone; the header matches `database_tasks.rb:308`.
- [ ] The missing-table arm matches `Kernel.abort` semantics (`:304`), not a
      plain throw, or the deviation is cited at the call site as language-forced.
- [ ] `packages/activerecord-cli/src/db-migrate-status.test.ts` keeps its test
      names and passes.
