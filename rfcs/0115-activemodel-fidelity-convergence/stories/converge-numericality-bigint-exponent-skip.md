---
title: "converge-numericality-bigint-exponent-skip"
status: ready
updated: 2026-08-24
rfc: "0115-activemodel-fidelity-convergence"
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

`packages/activemodel/src/validations/numericality-validation.test.ts:434-447`
carries `it.skip("validates numericality with exponent number")` behind a
`PERMANENT-SKIP` comment. It is the last remaining gap in activemodel's
`parity:test` numbers (`validations/numericality_validation_test.rb` 40 OK / 1
Skip / 41 total; the package sits at 962/963 after RFC 0115's
`port-user-input-in-time-zone-and-close-the-activemodel-test-gap`).

Rails (`vendor/rails/activemodel/test/cases/validations/numericality_validation_test.rb`)
asserts that with `less_than_or_equal_to: 10_000_000_000_000_000`, the string
`"10000000000000001"` is **invalid**. Ruby's `String#to_i` answers an
arbitrary-precision Integer, so `10000000000000001 > 10_000_000_000_000_000`.
Every JS number is a double: both sides round to `1e16`, the comparison is
less-than-or-equal, and the record validates.

The committed skip reason calls this PERMANENT. On re-read that claim is
overstated — JS _does_ have `BigInt`, and the reason itself names the shape that
would work ("a BigInt-typed parse pipeline through Comparability"). What makes it
non-trivial is blast radius, not impossibility: `parse_as_number` /
`Comparability` feed every numericality option (`greater_than`, `odd?`,
`only_integer`, the float arms), so carrying a BigInt for an integer-valued
string has to leave all the float comparisons untouched.

Do **not** reword the skip reason. Either converge it or `tasks block` it with
the specific blocker (CLAUDE.md: "converge or block, never re-justify").

## Acceptance criteria

- The numericality parse path carries exact integer arithmetic for an
  integer-valued string whose magnitude exceeds 2^53, without disturbing the
  float arms — every existing numericality test stays green.
- `it.skip("validates numericality with exponent number")` becomes a live `it`,
  with its name unchanged, and passes.
- The `PERMANENT-SKIP` comment block is deleted (not reworded).
- `parity:test` shows activemodel at **963/963**, `numericality_validation_test.rb`
  at Skip=0 Miss=0. State before/after in the PR body.
