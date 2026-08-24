---
title: "Map assert_predicate onto a kind a vitest matcher can produce"
status: ready
updated: 2026-08-24
rfc: "0122-arel-assertion-parity"
cluster: null
packages: []
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

The canonical assertion kinds `predicate` / `notPredicate` are reachable from
the Ruby side but from NO vitest matcher, so every Rails `assert_predicate` is a
permanent kind mismatch no matter how faithfully the trails test is written.

`scripts/test-compare/assertion-kinds.ts`:

- `RAILS_MAP` maps `assert_predicate: "predicate"`,
  `assert_not_predicate` / `refute_predicate: "notPredicate"` (lines ~101-103).
- `TRAILS_MAP` has no entry producing either kind (lines ~117-144).

Surfaced while converging `attribute-assertion-parity` (#7003). Rails
`attribute_test.rb:1136,1149` asserts
`assert_predicate table, :able_to_type_cast?`; the faithful trails spelling is
`expect(table.isAbleToTypeCast()).toBeTruthy()`, which normalizes to `truthy`
and still mismatches `predicate`. Two tests in
`packages/arel/src/attributes/attribute.test.ts` were left unconverged for
exactly this reason ("type casts when given an explicit caster", "does not type
cast SqlLiteral nodes") — they are the only residue in that file.

Note the sibling arm already converges: `assert_not` maps to `falsy`, which
`toBeFalsy` matches, and `attribute_test.rb:1129`'s
`assert_not table.able_to_type_cast?` converged cleanly in #7003.

## Converged shape

Minitest defines `assert_predicate obj, :pred?` as `assert obj.pred?` and
`refute_predicate` as `refute obj.pred?` (minitest/assertions.rb) — the same
truthiness assertion `assert` / `assert_not` make, just with the receiver and
message spelled apart. So map them onto the kinds that already carry that
meaning:

```ts
assert_predicate: "truthy",
assert_not_predicate: "falsy",
refute_predicate: "falsy",
```

and drop `predicate` / `notPredicate` from `CanonicalKind` once no mapping
produces them. `expect(x.pred()).toBeTruthy()` then matches, and the receiver /
message split is not information the kind histogram carries anyway.

Verify the `NEGATION` table (assertion-kinds.ts ~line 51-63) stays consistent
after the removal — it currently pairs `predicate`/`notPredicate`.

## Acceptance criteria

- `assert_predicate` / `assert_not_predicate` / `refute_predicate` normalize to
  a kind a vitest matcher can produce.
- `scripts/test-compare/assertion-kinds.test.ts` covers both arms.
- The two `attribute.test.ts` type-casting tests above are converged to
  `toBeTruthy()` and no longer appear in `pnpm parity:test --assertions
--missing --package arel`.
- Report the effect on all five marked packages; `assertion-mismatch-mark.json`
  is tightened for every package that moves DOWN, and no package moves up.
- `pnpm parity:test:assertions` green.
