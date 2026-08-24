---
title: "Map Minitest spec forms, must_be_like and assert_edge in assertion-kinds"
status: ready
updated: 2026-08-24
rfc: "0122-arel-assertion-parity"
cluster: null
packages: ["arel"]
deps: []
deps-rfc: []
est-loc: 220
priority: 1
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`normalizeRailsKind` (`scripts/test-compare/assertion-kinds.ts:150-160`) folds
Minitest spec forms with a prefix rewrite: `must_<suffix>` → `assert_<suffix>`,
`wont_<suffix>` → `refute_<suffix>`. That is right for `must_equal` and wrong
for the entire `must_be_*` family — `must_be_kind_of` rewrites to
`assert_be_kind_of`, which is not a Minitest assertion and is not a key in
`RAILS_MAP`, so it normalizes to `null` and lands in `unmapped`.

arel is the only suite this hits at scale: `vendor/rails/activerecord/test/cases/arel/helper.rb:29`
defines `Arel::Spec < Minitest::Spec`, so its tests assert through spec
expectations. Measured unmapped tokens across arel's 590 kind mismatches:

- `must_be_like` × 326 — arel's own helper, `helper.rb:10-13`: it
  `gsub(/\s+/, " ").strip`s BOTH operands and delegates to `must_equal`, so it
  is canonical kind `equal`.
- `must_be_kind_of` × 52 → `instanceOf` (spec form of `assert_kind_of`).
- `assert_edge` × 11 — `vendor/rails/activerecord/test/cases/arel/visitors/dot_test.rb:13-15`,
  a file-local helper wrapping `assert_match(/->.*label="#{edge_name}"/, …)`
  → `match`.
- `must_be_nil` × 5 → `nil`; `wont_be_same_as` × 4 → `notSame`.

Mapping `must_be_like` also makes 326 assertions **value**-comparable for the
first time, and `assertion-values.ts` compares string tokens verbatim
(`collectSide`, `assertion-values.ts:83-102`). The Ruby side is a `%{ … }`
heredoc that keeps its indentation; the ported TS literal does not. That is
76 of the resulting 155 value mismatches, and it is formatting, not divergence —
whitespace-insensitivity is a property of the Rails-side helper, so the fold
must be decided from the Rails kinds and applied to BOTH sides.

The remaining 79 value mismatches are real content divergence
(`to_sql_test.rb`'s `should visit_Float` asserts `"products"."price" = 2.14`
where `to-sql.test.ts` asserts `1.5`). They are too many to bundle here and
cannot be converged before this lands: today the Rails side of a `must_be_like`
pair contributes nothing to the histogram, so a test fix is unmeasurable. RFC
0122 therefore authorises a **one-time, dimension-scoped** correction of arel's
`value` mark, 17 → 79 — the old 17 measured almost nothing. It does not extend
to any other package, dimension, or story.

## Acceptance criteria

- `assertion-kinds.ts` gains a `SPEC_MAP` consulted by `normalizeRailsKind`
  alongside `RAILS_MAP`, covering `must_be_nil`, `wont_be_nil`, `must_be_empty`,
  `wont_be_empty`, `must_be_kind_of`, `must_be_instance_of`, `must_be_same_as`,
  `wont_be_same_as`, `must_be`, `must_be_close_to`, `must_be_within_delta`,
  plus `must_be_like` and `assert_edge`. Each arel-local helper entry carries a
  comment citing the Ruby `file:line` that defines it.
- `assertion-values.ts` folds runs of whitespace in an `equal`-kind `s:` token
  on both sides when the Rails kinds include `must_be_like`, with a comment
  citing `helper.rb:10-13`.
- Unit coverage in `assertion-kinds.test.ts` and `assertion-values.test.ts` for
  the new spec forms and the whitespace fold.
- `pnpm parity:test -- --assertions` reports, and the PR body states, the
  before/after for ALL five gated packages. Verified locally:
  arel 178/416/79 (from 178/590/17 measured, 180/591/17 marked),
  activemodel 304/**446**/59 (kind was 448),
  activerecord 1940/3924/36 and activesupport 864/1213/104 both unchanged.
- `scripts/test-compare/assertion-mismatch-mark.json`: arel becomes
  `{ assertionCount: 178, kind: 416, value: 79 }`; activemodel `kind` tightens
  448 → 446. No other entry moves, and no other entry rises.
- `pnpm parity:test:assertions` is green.
