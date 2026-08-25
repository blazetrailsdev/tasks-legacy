---
title: "SQLite3Adapter does not expand a relative database against the root"
status: draft
updated: 2026-07-31
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
  - "activerecord-cli"
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

Rails expands a relative SQLite `database` at the adapter layer:
`sqlite3_adapter.rb:49` (`File.expand_path(config.database, Rails.respond_to?(:root) ? Rails.root : nil)`)
and `:113-114` (`@config[:database] = File.expand_path(@config[:database], Rails.root) if defined?(Rails.root)`).

trails' SQLite3 adapter does no such expansion, so a relative
`db/development.sqlite3` opens against the process cwd. PR #5735 worked around
this in `activerecord-cli` only: `normalizeSqlitePaths`
(`packages/activerecord-cli/src/environment.ts`) rewrites every sqlite config at
`config/database.ts` load time, resolving against `DatabaseTasks.root`. Anything
that establishes a sqlite connection outside the CLI still diverges.

The task layer already resolves against the root independently
(`packages/activerecord/src/tasks/sqlite-database-tasks.ts:261-270`), so there
are now two root-resolution sites and one adapter that has none.

## Acceptance criteria

- [ ] SQLite3Adapter expands a relative `database` against the trails root
      analogue at connect time, matching sqlite3_adapter.rb:49,113-114
      (`:memory:` and `file:` URIs excluded, as in Rails).
- [ ] `normalizeSqlitePaths` and its call site in `db-helpers.ts` are retired
      once the adapter covers it, or the remaining CLI-only need is justified.
- [ ] The activerecord-cli sqlite happy-path E2E still passes, and a relative
      sqlite path still resolves against the project root.
