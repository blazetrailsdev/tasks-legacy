---
title: "Remove bespoke duplicate where_sql describe blocks in select-manager.test.ts"
status: ready
updated: 2026-07-27
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "arel"
deps: []
deps-rfc: []
est-loc: 30
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`packages/arel/src/select-manager.test.ts` has TWO `describe("where_sql")`
blocks. The one at ~line 782 is bespoke: `gives me back the where sql` and
`joins wheres with AND` there never call `whereSql()` at all — they assert
`mgr.constraints.length` and `toSql()).toContain("AND")`. The real ports of
those Rails tests (`vendor/rails/activerecord/test/cases/arel/select_manager_test.rb:948-964`)
landed in the second block (~line 1401) in #5180, so the first block is now a
duplicate under the same test names, weakening what `parity:test` matches.
A third bespoke squatter in that block (`handles database-specific statements`,
asserting `FOR UPDATE`) was deleted by #5201; the remaining two were left in
place to keep that PR scoped.

There is also a bespoke `describe("where_sql")` at ~line 1213 containing a
third copy of `gives me back the where sql`.

## Acceptance criteria

- [ ] The bespoke `describe("where_sql")` blocks are removed; only the block
      holding the faithful Rails ports remains.
- [ ] Each Rails `where_sql` test name appears exactly once in the file.
- [ ] parity:test delta non-negative.
