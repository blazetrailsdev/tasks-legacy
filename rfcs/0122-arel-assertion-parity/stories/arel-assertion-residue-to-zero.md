---
title: "arel-assertion-residue-to-zero"
status: ready
updated: 2026-08-25
rfc: "0122-arel-assertion-parity"
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
closed-reason: null
---

## Context

Continues `arel-assertion-mark-to-zero` (RFC 0122), which burnt arel's
assertion dimensions from `39 / 81 / 11` down to `32 / 53 / 7` and tightened
`scripts/test-compare/assertion-mismatch-mark.json` accordingly. The remainder
did not fit that PR's LOC ceiling.

Measure the residue with:

```bash
pnpm parity:test -- --assertions --missing --package arel
```

What is left, from that report:

- **Two extractor gaps, not port drift.** `nodes/infix_operation_test.rb ›
construct` reports `equal rails [n:1, n:2, s:] vs trails [n:1, n:2, s:+]` and
  `nodes/unary_operation_test.rb › construct` reports `[n:1, s:] vs [n:1, s:-]`.
  Rails asserts `assert_equal :+, operation.operator`
  (`vendor/rails/activerecord/test/cases/arel/nodes/infix_operation_test.rb:9`,
  `unary_operation_test.rb:9`); the Ruby assertion extractor yields an EMPTY
  literal for an operator-named Symbol (`:+`, `:-`) where the trails side
  carries `"+"` / `"-"`. Fix in `scripts/test-compare/` — never by editing the
  test to match the extractor.
- **Real value drift still open**, each a one-line convergence to the Rails
  literal: `nodes/sql_literal_test.rb › makes a distinct node`,
  `nodes/over_test.rb › should use empty definition`,
  `collectors/sql_string_test.rb › compile`,
  `nodes/fragments_test.rb › is not equal with different values`,
  `nodes/bound_sql_literal_test.rb › is not equal with different components`
  (the last two assert a boolean where Rails asserts a `uniq.size`).
- **~53 kind mismatches and ~32 count mismatches** across
  `select_manager_test.rb`, `attributes/attribute_test.rb`,
  `visitors/postgres_test.rb`, `nodes/case_test.rb`, `nodes/sql_literal_test.rb`,
  `nodes/select_core_test.rb`, `nodes/node_test.rb`, `nodes_test.rb`,
  `nodes/bind_param_test.rb`, `nodes/casted_test.rb`, `nodes/filter_test.rb`,
  `nodes/over_test.rb`, `nodes/equality_test.rb`, the clone tests on
  `{delete,insert,update,select}_statement_test.rb`, and
  `collectors/{bind,composite}_test.rb`. The recurring shapes are the same
  ones the first pass converged: a Rails `array.uniq.size` assertion ported as a
  `hash()` / field comparison (use `uniq` from
  `packages/arel/src/test-helpers/uniq.ts`), `assert_predicate` ported as
  `toBe(true)` rather than `toBeTruthy()`, `must_be_kind_of` ported as an extra
  `toBeInstanceOf` alongside the Rails `assert_equal`, and `assert_not_same`
  ported as `not.toBe`.
- **425 extra (TS only) assertions.** Trails-only rigour inside a mirrored test
  moves to a `.trails.test.ts` sibling (see
  `packages/arel/src/nodes/unary-operation.trails.test.ts`, created by the first
  pass). Nothing is deleted to move a number.

## Acceptance criteria

- [ ] `pnpm parity:test -- --assertions --package arel` reports
      `0 assertion-count-mismatch, 0 assertion-kind-mismatch, 0 assertion-value-mismatch`.
- [ ] `assertion-mismatch-mark.json`'s arel entry reads
      `{ assertionCount: 0, kind: 0, value: 0 }` and no other package's mark rises.
- [ ] The two operator-Symbol value mismatches are fixed in the extractor with a
      before/after across all packages, not by editing a test.
- [ ] No test name is renamed or reworded.
- [ ] `pnpm parity:test:assertions` green.
