---
title: "arel-duplicate-where-sql-describe-blocks"
status: closed
updated: 2026-08-25
rfc: "0124-arel-surfaced-deviations"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: "Duplicate of dedupe-bespoke-where-sql-describe-blocks (same file, same two where_sql describe blocks), which predates it"
---

## Context

Surfaced in review of PR #7054 (`arel-weakened-test-body-restoration-sweep`).

`packages/arel/src/select-manager.test.ts` still carries two top-level
`describe("where_sql", ...)` blocks after #7054 deleted a third:

- the faithful port of `vendor/rails/activerecord/test/cases/arel/select_manager_test.rb:948-966`,
  using `mustBeLike` on `manager.whereSql()!.value`;
- a later duplicate asserting `instanceof Nodes.SqlLiteral` plus the exact SQL,
  which also hosts a trails-only "handles database-specific statements" case.

Rails has exactly one `where_sql` describe. Two blocks with the same Rails test
names inside them means `parity:test` matches one and the other is dead weight
that can silently drift from the Rails body.

PR #7054 deleted only the one weak duplicate block its sweep targeted (projections
/ projections= / where_sql / froms / orders / exists) and left this pair, since
resolving it means deciding where the trails-only case lives.

## Acceptance criteria

- One `describe("where_sql")` in `select-manager.test.ts`, with the body Rails
  has (`select_manager_test.rb:948-966`); test names unchanged.
- The trails-only "handles database-specific statements" case is kept, moved to
  the trails-only sibling file if it does not belong under a Rails describe.
- Sweep the same file for any other duplicated Rails describe/test name while
  there.
- `pnpm parity:test` delta non-negative; `pnpm vitest run packages/arel` green.
