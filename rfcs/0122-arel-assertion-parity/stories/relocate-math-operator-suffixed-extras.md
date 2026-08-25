---
title: "relocate-math-operator-suffixed-extras"
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

`packages/arel/src/attributes/math.test.ts` holds 55 tests: the 10 that mirror
`vendor/rails/activerecord/test/cases/arel/attributes/math_test.rb` (whose Rails
names extract to the interpolation-stripped `"average should be compatible with "`
— `math_test.rb:9,46`, `it "... #{math_operator}"`), plus 45 trails-only
exhaustive cases named with an explicit operator suffix
(`"average should be compatible with *"`, `… "/"`, `… "+"`, `… "<<"`, one per
operator × the 5 receivers).

`parity:test` counts those 45 as `extra (TS only)`. Per RFC 0122 they belong in a
`.trails.test.ts` sibling, not in the mirrored file — a reviewer on PR #7029 read
them as duplicates of the mirrored tests, which is exactly the confusion the
relocation rule exists to prevent.

Out of scope for #7029 (`table-and-factory-assertion-parity`): the move is ~400
counted LOC on top of that PR's ~560, over the ceiling.

## Acceptance criteria

- The 45 operator-suffixed cases move verbatim to
  `packages/arel/src/attributes/math.trails.test.ts` with a header noting they
  have no Rails counterpart.
- `math.test.ts` keeps exactly the 10 mirrored tests; no test name is renamed.
- `pnpm parity:test -- --assertions --package arel` shows math_test.rb still
  10/10 matched with zero count/kind/value mismatches, and arel's marks in
  `scripts/test-compare/assertion-mismatch-mark.json` are unchanged or tightened.
