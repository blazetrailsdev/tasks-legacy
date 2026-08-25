---
title: "db_warnings_action is resolved per adapter, not once at config time"
status: claimed
updated: 2026-08-25
rfc: "0112-one-rails-thing-n-trails-things"
cluster: duplicate-bodies
packages: []
deps: []
deps-rfc: []
est-loc: 90
pr: null
claim: "2026-08-25T17:22:41Z"
assignee: "current-attributes-port-body"
blocked-by: null
closed-reason: null
---

## Context

Rails resolves `db_warnings_action` **once, at config time**: the setter
`ActiveRecord.db_warnings_action=`
(`vendor/rails/activerecord/lib/active_record.rb:236-252`) turns the symbol
(`:ignore` / `:log` / `:raise` / `:report`) or callable into a single Proc held
in `ActiveRecord.db_warnings_action`. Every adapter's `handle_warnings` then
does nothing but `ActiveRecord.db_warnings_action.call(warning)` —
PostgreSQL's is
`vendor/rails/activerecord/lib/active_record/connection_adapters/postgresql/database_statements.rb:216-223`.

trails instead stores the raw symbol on the adapter class
(`AbstractAdapter.dbWarningsAction`, `connection-adapters/abstract-adapter.ts:814`)
and re-implements the symbol → behavior mapping inside **each adapter's**
warning handler:

- `connection-adapters/postgresql/database-statements.ts` (`handleWarnings`,
  landed in PR #6330 — dispatches all four arms)
- `connection-adapters/mysql2-adapter.ts` (`_handleWarningsOn`, ~:1767-1806 —
  missing the `:report` arm, see [[mysql2-handle-warnings-report-arm]])

Duplicating the mapping is what let the two adapters drift apart on what
`dbWarningsAction = "report"` does; the MySQL gap is an instance of this root
divergence, not the cause.

## Converged shape

Resolve the symbol into a callable once, where Rails does, and leave every
`handleWarnings` as a single `.call(warning)`:

- Hold the resolved callable on the `ActiveRecord` namespace object, set by a
  `dbWarningsAction` setter that mirrors `active_record.rb:236-252` (including
  `:report` → `Rails.error.report(warning, handled: true)`, which trails spells
  `ActiveSupport.errorReporter.report(warning, { handled: true })`).
- Keep the reader answering the resolved callable so a live change between
  queries takes effect, as it does in Rails.
- Each adapter's `handle_warnings` port then reduces to Rails' loop:
  `warning_ignored?` → `warning.sql = …` → `.call(warning)`.

Note `AbstractAdapter.dbWarningsAction` is currently read as a _class_ attribute
by both adapters and by `support/with-db-warnings-action.ts`; converging the
storage location has to keep those readers working (or move them).

## Acceptance criteria

- [ ] The symbol → behavior mapping exists in exactly one place, mirroring
      `active_record.rb:236-252`.
- [ ] PostgreSQL's and mysql2's warning handlers contain no per-arm branching —
      each just invokes the resolved callable.
- [ ] All four arms plus a user-supplied callable behave identically on both
      adapters; the existing PG cases in
      `adapters/postgresql/postgresql-adapter.test.ts`
      (`ignores warnings when behaviour ignore`, `logs warnings when behaviour
log`, `raises warnings when behaviour raise`, `reports when behaviour
report`, `warnings behaviour can be customized with a proc`) stay green.
- [ ] `support/with-db-warnings-action.ts` still scopes the action for tests.
- [ ] `pnpm parity:api:calls` / `pnpm parity:api:extra --package activerecord` clean; no new
      baseline rows.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
