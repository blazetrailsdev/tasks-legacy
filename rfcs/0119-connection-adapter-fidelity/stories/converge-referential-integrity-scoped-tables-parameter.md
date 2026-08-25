---
title: "converge-referential-integrity-scoped-tables-parameter"
status: ready
updated: 2026-08-25
rfc: "0119-connection-adapter-fidelity"
cluster: null
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

`disableReferentialIntegrity`
(`packages/activerecord/src/connection-adapters/postgresql/referential-integrity.ts:66-92`)
still takes a trails-only optional `scopedTables?: string[]` parameter. Rails'
`disable_referential_integrity` is zero-arg in every adapter —
`vendor/rails/activerecord/lib/active_record/connection_adapters/postgresql/referential_integrity.rb:7`,
`sqlite3_adapter.rb:255`, `abstract_mysql_adapter.rb:212`.

The earlier story `converge-referential-integrity-zero-arg-shape`
(RFC 0060, merged as #4545) closed on its "converge OR document" arm: it
documented the parameter rather than removing it — so the doc comment now cites a landed story as if the
convergence were still pending. Per CLAUDE.md a documented deviation is debt,
not permission, so the convergence still owes.

Deviation surface (unchanged since #4545):

- `referential-integrity.ts` — `scopedTables?: string[]` on the exported fn.
- `abstract/database-statements.ts` — `DatabaseStatementsHost` mirrors it.
- `database-statements.ts` `truncateTables` / `insertFixturesSet` call sites
  pass the scoped set.

## Acceptance criteria

- Hoist the fixture-load / reset flow to a single `disableReferentialIntegrity`
  block per fixture set, matching Rails' `insert_fixtures_set`
  (`vendor/rails/activerecord/lib/active_record/fixtures.rb:684`).
- Drop the `scopedTables` parameter and the host-interface mirror, restoring the
  zero-arg Rails shape.
- Update the `referential-integrity.ts` comment so it no longer names a landed
  story as pending.
