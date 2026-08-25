---
title: "Move the trails-only addForeignKey ifNotExists cases out of the Rails-mapped on-adapter test file"
status: ready
updated: 2026-08-25
rfc: "0113-branch-and-guard-parity"
cluster: invented-arm
deps: []
deps-rfc: []
est-loc: 40
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`packages/activerecord/src/connection-adapters/abstract/schema-statements-on-adapter.test.ts`
carries four trails-invented `addForeignKey` + `ifNotExists` cases (added
around lines 420-520) that have no counterpart in Rails'
`test/cases/migration/foreign_key_test.rb`. They assert the abstract
`add_foreign_key` guard at
`vendor/rails/activerecord/lib/active_record/connection_adapters/abstract/schema_statements.rb:1173-1181`,
and PR #5608 had to gate three of them per adapter (`skipIf`) once the
invented SQLite override guard was deleted (SQLite's override,
`sqlite3/schema_statements.rb:56-63`, has no such guard).

Because the file is a Rails-mapped `*.test.ts`, these cases count as
extra TS-only tests in `parity:test`. Per the convention, TS-only extras
belong in a sibling `*.trails.test.ts`.

## Acceptance criteria

- [ ] The four trails-only `addForeignKey`/`ifNotExists` cases move to a
      `*.trails.test.ts` sibling, keeping the existing per-adapter gating
      (`adapterType !== "sqlite"` for the abstract-guard cases, SQLite-only
      for the fall-through case).
- [ ] `parity:test` extra-test count does not increase; the moved cases
      still run and pass on all three adapters.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0113-branch-and-guard-parity`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
