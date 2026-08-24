---
title: "nodes-cluster-assertion-parity-tail"
status: ready
updated: 2026-08-24
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

The remainder of `nodes-cluster-assertion-parity` (RFC 0122). That story shipped
the equality-family convergence for `nodes/and`, `nodes/ascending`,
`nodes/descending`, `nodes/as` and `nodes/cte` (PR pending), which took arel's
assertion marks from `count 151 / kind 334 / value 58` to
`count 143 / kind 317 / value 57`. It hit the 700-LOC PR ceiling there.

The story's stated "44 assertion-kind mismatches" was measured before
`map-minitest-spec-assertion-forms` landed in
`scripts/test-compare/assertion-kinds.ts`; the real cluster tail is ~75 kind
mismatches, listed by
`pnpm parity:test -- --assertions --missing --package arel`.

The dominant class is unchanged and mechanical: many trails node tests are
placeholder ports that assert something _other_ than the Rails body — e.g.
`packages/arel/src/nodes/count.test.ts`'s `equality` describe constructs `Cte`
and `TableAlias` nodes where
`vendor/rails/activerecord/test/cases/arel/nodes/count_test.rb:26-34` builds two
`Count` nodes and asserts `assert_equal 1, array.uniq.size`. Port the Rails body.

Remaining files, by mismatch count (top first): `nodes/sql_literal_test.rb` (5),
`nodes/case_test.rb` (5), `nodes/unary_operation_test.rb` (4),
`nodes/window_test.rb` (3), `nodes/sum_test.rb` (3), `nodes/select_core_test.rb`
(3), `nodes/infix_operation_test.rb` (3), `nodes/fragments_test.rb` (3),
`nodes/extract_test.rb` (3), `nodes/equality_test.rb` (3), `nodes/count_test.rb`
(3), then a 1–2 tail across `nodes/*`, `attributes_test.rb`, `nodes_test.rb`
and `collectors/*`.

Helpers already hoisted and to be reused, not redefined:
`packages/arel/src/test-helpers/must-be-like.ts` (helper.rb:10-13) and
`packages/arel/src/test-helpers/uniq.ts` (Ruby `Array#uniq`, which the
`assert_equal 1, array.uniq.size` bodies lean on).

## Acceptance criteria

- Every remaining mismatch in `packages/arel/src/nodes/*.test.ts`,
  `attributes.test.ts`, `nodes.test.ts` and `collectors/*.test.ts` is triaged
  into exactly one bucket and handled, per the parent story: real divergence
  (mirror the Ruby), legitimate trails-only extra (move to a `.trails.test.ts`
  sibling), or a tooling gap (fix `scripts/test-compare/assertion-kinds.ts`
  with a one-line justification citing both sides' semantics, and report the
  effect on all five packages' marks).
- No test name is renamed or reworded.
- The touched tests pass under `pnpm vitest run packages/arel/src`.
- `scripts/test-compare/assertion-mismatch-mark.json`'s arel entry is tightened
  on each of the three dimensions by the amount converged; the other four
  packages' marks are unchanged.
- `pnpm parity:test:assertions` is green.
