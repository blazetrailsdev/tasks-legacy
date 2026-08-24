---
rfc: "0122-arel-assertion-parity"
title: "arel assertion parity to zero"
status: active
created: 2026-08-24
updated: 2026-08-24
owner: "@your-handle"
packages:
  - "arel"
clusters: []
priority: 3
---

## Summary

Drive `arel`'s **assertion** parity to zero on all three ratchet dimensions.
arel's test dimension finished at 707/707 (100%) and no active RFC owns the
package since `0117-arel-extra-surface-burndown` closed on 2026-08-24.

The assertion dimension is the worst in the repo, and by a wide margin:

| package       | matched | count-mm | kind-mm | kind rate | value-mm |
| ------------- | ------- | -------- | ------- | --------- | -------- |
| **arel**      | **707** | **178**  | **590** | **83.5%** | 17       |
| activerecord  | 8236    | 1942     | 3924    | 47.6%     | 37       |
| activesupport | 2554    | 863      | 1212    | 47.5%     | 104      |
| activemodel   | 961     | 304      | 448     | 46.6%     | 59       |

## Why arel is the outlier: it is the only minitest **spec**-style suite

`vendor/rails/activerecord/test/cases/arel/helper.rb:29` defines
`Arel::Spec < Minitest::Spec`, so arel's tests assert through the spec
expectations (`_(x).must_be_like`, `must_be_kind_of`, `wont_be_same_as`) rather
than through `assert_*`. Every other package uses `ActiveSupport::TestCase`.

`normalizeRailsKind` (`scripts/test-compare/assertion-kinds.ts:150-160`) handles
spec forms with a prefix rewrite — `must_<suffix>` → `assert_<suffix>`. That is
correct for `must_equal` and wrong for the whole `must_be_*` family:
`must_be_kind_of` rewrites to `assert_be_kind_of`, which is not a Minitest
assertion and is not in `RAILS_MAP`, so it normalizes to `null` and lands in
`unmapped`. Measured unmapped tokens across arel's 590 kind mismatches:

| token             | tests | what it actually is                                                                                  |
| ----------------- | ----- | ---------------------------------------------------------------------------------------------------- |
| `must_be_like`    | 326   | arel-local helper, `helper.rb:10-13` — whitespace-squeezes both operands then `must_equal` → `equal` |
| `must_be_kind_of` | 52    | spec form of `assert_kind_of` → `instanceOf`                                                         |
| `assert_edge`     | 11    | `visitors/dot_test.rb:13-15`, wraps `assert_match` → `match`                                         |
| `must_be_nil`     | 5     | spec form of `assert_nil` → `nil`                                                                    |
| `wont_be_same_as` | 4     | spec form of `refute_same` → `notSame`                                                               |

## Triage split (measured, not estimated)

Of the 590 kind mismatches:

- **174 are tooling false positives** (29%) — cleared by mapping the spec forms
  above. Measured by patching `assertion-kinds.ts` and re-running
  `pnpm parity:test -- --assertions`: arel kind 590 → **416**, activemodel
  448 → 446, activerecord and activesupport unchanged. **No package's numbers
  rose.**
- **416 are real divergence** (71%). The dominant class, 223 rows, is a Rails
  full-string `must_be_like` ported as a substring `toContain` — often split
  into two, which is also most of the 178 count mismatches. Example:
  `select_manager_test.rb:15-23` asserts
  `must_be_like %{ SELECT id FROM "users" }`; `select-manager.test.ts:35`
  asserts `expect(mgr.toSql()).toContain("SELECT id")`. A further 62 rows are
  `instanceOf rails 0 vs trails 1` — extra `toBeInstanceOf` with no Rails
  counterpart.

## The value dimension needs a one-time mark correction

Mapping `must_be_like` makes 326 previously-invisible assertions
value-comparable for the first time, so arel's value count moves 17 → 155. Of
those 155, **76 are pure whitespace**: `must_be_like` squeezes and strips both
operands, the Ruby heredoc keeps its indentation and the ported TS literal does
not. `assertion-values.ts` compares string tokens verbatim, so it must fold
whitespace for that helper — on both sides, since the property belongs to the
Rails-side helper. With that fold, arel measures exactly **79**.

Those 79 are real (`should visit_Float` asserts `1.5` where
`to_sql_test.rb` asserts `"products"."price" = 2.14`). They are far too many to
bundle into the tooling PR, and they cannot be converged before it: under
today's mapping the Rails side of a `must_be_like` pair contributes nothing to
the histogram, so a test fix is unmeasurable until the mapping lands.

So the first story raises arel's `value` mark 17 → 79, **once**, as a
correction: the old 17 measured almost nothing because 326 of the package's
assertions were unmapped. This is a deliberate exception to the only-shrink
rule, approved for this RFC and this dimension only. It does not extend to
`assertionCount` or `kind` (both of which move DOWN in the same PR), to any
other package, or to any later story here — every story after the first is
strictly only-shrink.

## Mark movement

`scripts/test-compare/assertion-mismatch-mark.json`, arel:

````text
start   { assertionCount: 180, kind: 591, value: 17 }
after tooling story   { assertionCount: 178, kind: 416, value: 79 }
end     { assertionCount: 0,   kind: 0,   value: 0 }
```text

activemodel's `kind` also tightens 448 → 446 in the tooling story (the same
spec-form rules fire on two of its tests).

## Constraints every story here inherits

- **NEVER rename or reword a test name.** Names are how `parity:test` matches.
  If a test's behaviour does not fit its name, the implementation changes.
- A mapping rule is not a way to make a real divergence disappear. Each rule
  carries a one-line justification a reviewer can check against both sides'
  semantics, citing the Ruby `file:line` that defines the helper.
- `assertion-kinds.ts` moves **every** package's numbers. Any change to it
  reports its effect on all five marks, before and after.
- Apart from the single documented correction above, the mark file is
  only-shrink. Edit the numbers you converged; never reseed.

## Remaining kind mismatches by file (post-tooling, the burndown)

```text
76  visitors/to_sql_test.rb        14  table_test.rb          5  nodes/case_test.rb
63  select_manager_test.rb         12  visitors/mysql_test.rb 5  nodes/ascending_test.rb
55  attributes/attribute_test.rb   12  insert_manager_test.rb 5  nodes/descending_test.rb
35  visitors/postgres_test.rb       10 attributes/math_test.rb 4 visitors/sqlite_test.rb
16  visitors/dot_test.rb             9 update_manager_test.rb  ~40 nodes/* tail
```text

## Triage rule for the per-file stories

Every mismatch lands in exactly one bucket:

- **real divergence** — the trails test asserts something different from the
  Rails test. Fix the test to mirror the Ruby. A `toContain` where Rails
  asserts a full string becomes a `toEqual` on the whitespace-squeezed SQL.
- **legitimate trails-only extra** — an assertion with no Rails counterpart.
  Move it to a `.trails.test.ts` sibling. Do not delete rigour, and do not
  leave it inflating a mirrored test's count.
- **tooling false positive** — if a per-file story turns up a further mapping
  or extractor gap, it goes back to the tooling story's file with its own
  justification, not into a per-file workaround.

## End condition

`pnpm parity:test -- --assertions --package arel` reports
`0 assertion-count-mismatch, 0 assertion-kind-mismatch, 0 assertion-value-mismatch`
and the arel entry in `assertion-mismatch-mark.json` reads `0 / 0 / 0`, with the
other four packages' marks no higher than they are today.
````
