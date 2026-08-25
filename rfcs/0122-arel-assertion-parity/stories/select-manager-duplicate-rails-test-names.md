---
title: "select-manager.test.ts duplicates 12 Rails test names with weaker trails-only copies"
status: draft
updated: 2026-08-25
rfc: "0122-arel-assertion-parity"
cluster: null
packages: []
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

`packages/arel/src/select-manager.test.ts` carries a cluster of trails-only
tests that re-use Rails test names already mirrored elsewhere in the same file,
with weaker assertions. They are part of the file's 55 "extra (TS only)" rows in
`pnpm parity:test -- --assertions --package arel`.

Confirmed duplicate names (each `it` appears twice in the file, where
`vendor/rails/activerecord/test/cases/arel/select_manager_test.rb` defines the
name exactly once):

`adds a lock node`, `can be aliased`, `copies where`,
`gives me back the where sql`, `joins wheres with AND`, `overwrites projections`,
`reads projections`, `returns inner join sql`, `returns outer join sql`,
`returns order clauses`, `should hand back froms`,
`takes a range frame, current row`.

(Names Rails itself repeats across different `describe` blocks — `chains`,
`noops on nil`, `responds to join`, `should add an offset`, `sets the quantifier`,
`takes multiple args` — are NOT in scope; those duplicates are correct.)

The first occurrence of each was converged to the Rails body in PR #7025 (RFC
0122 `select-manager-assertion-parity`); the second is the weaker leftover, e.g.
`should hand back froms` at the Rails-mirrored `expect(relation.froms).toEqual([])`
versus a trails-only copy asserting `expect(manager.source).toBeDefined()` under
the same name. Leaving both means one Rails test name maps to two TS tests, and
`parity:test` pairs the first while counting the second as an extra.

## Converged shape

For each name above: keep the Rails-mirroring occurrence, and for the leftover
either delete it (where the mirrored test strictly covers it — the common case)
or, where it genuinely asserts something Rails does not, move it to
`select-manager.trails.test.ts` under a name that is NOT a Rails test name.
Do not reword the mirrored test's name.

## Acceptance criteria

- No Rails test name from `select_manager_test.rb` appears twice in
  `select-manager.test.ts`.
- Any retained trails-only rigour lives in `select-manager.trails.test.ts`.
- arel's "extra (TS only)" count for this file drops accordingly; the three
  `assertion-mismatch-mark.json` arel dimensions do not increase.
- `pnpm vitest run packages/arel/src/select-manager.test.ts` green.
