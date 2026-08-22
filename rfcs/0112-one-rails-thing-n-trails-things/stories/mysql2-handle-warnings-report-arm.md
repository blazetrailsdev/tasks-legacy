---
title: "mysql2-handle-warnings-report-arm"
status: ready
updated: 2026-08-22
rfc: "0112-one-rails-thing-n-trails-things"
cluster: duplicate-bodies
packages: []
deps: []
deps-rfc: []
est-loc: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`Mysql2Adapter#_handleWarningsOn`
(`packages/activerecord/src/connection-adapters/mysql2-adapter.ts:1767-1806`)
dispatches the configured `dbWarningsAction` for the `ignore` / `log` / `raise`
/ callable arms but not for `:report`, leaving a `TODO(report)` at
`mysql2-adapter.ts:1802-1803`.

Rails has no per-adapter dispatch at all: `handle_warnings`
(`vendor/rails/activerecord/lib/active_record/connection_adapters/abstract_adapter.rb`,
and PostgreSQL's override at
`connection_adapters/postgresql/database_statements.rb:216-223`) only calls
`ActiveRecord.db_warnings_action.call(warning)`. The symbol → behavior mapping,
including `:report` → `Rails.error.report(warning, handled: true)`, is baked
into the Proc by `ActiveRecord.db_warnings_action=` in
`vendor/rails/activerecord/lib/active_record.rb:236-252`.

PR #6330 converged PostgreSQL's `handleWarnings`
(`packages/activerecord/src/connection-adapters/postgresql/database-statements.ts`),
which now dispatches all four arms, `:report` included, via
`ActiveSupport.errorReporter.report(warning, { handled: true })`. MySQL is the
remaining adapter with the gap, so the two adapters disagree on what
`dbWarningsAction = "report"` does. Surfaced by review on PR #6330 as
out-of-scope for that PG-only story.

Note the deeper divergence worth considering while here: trails stores the
symbol on the adapter class and each adapter re-implements the mapping, where
Rails resolves it once at config time into a single global Proc. Converging the
mapping to one place would remove the class of bug this story is fixing an
instance of.

## Acceptance criteria

- [ ] `dbWarningsAction = "report"` reports through `ActiveSupport.errorReporter`
      with `handled: true` on the mysql2 adapter, matching PostgreSQL and
      `active_record.rb:248-249`.
- [ ] The `TODO(report)` comment at `mysql2-adapter.ts:1802-1803` is deleted.
- [ ] A test covers the `report` arm on mysql2, mirroring the PG case
      `reports when behaviour report`
      (`packages/activerecord/src/adapters/postgresql/postgresql-adapter.test.ts`).
- [ ] `pnpm parity:api:calls` and `pnpm parity:api:extra --package activerecord` stay clean;
      no new baseline rows.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
