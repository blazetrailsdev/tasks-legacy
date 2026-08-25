---
title: "Type::Float#cast answers nil where Ruby String#to_f answers 0.0"
status: draft
updated: 2026-08-14
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activemodel"
deps: []
deps-rfc: []
est-loc: 150
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while porting `type/float_test.rb`'s assertions in PR #6519 (RFC 0105).
The assertion port is blocked on this deviation: converging the assertions alone
reds the test.

Rails `Type::Float` casts a String through Ruby's `String#to_f`, which parses a
leading numeric run and answers `0.0` when there isn't one — it never answers
`nil` for a non-blank string. `test_type_cast_float_from_invalid_string`
(`vendor/rails/activemodel/test/cases/type/float_test.rb:13-19`) pins exactly
that:

```ruby
assert_nil type.cast("")
assert_equal 1.0, type.cast("1ignore")
assert_equal 0.0, type.cast("bad1")
assert_equal 0.0, type.cast("bad")
```

trails `FloatType.cast` answers `null` for any non-numeric string, so
`cast("bad")` is `null` where Rails is `0.0` and `cast("1ignore")` is `null`
where Rails is `1.0`. The existing trails test asserts the trails behaviour
(`type.cast("not-a-number")` -> `null`), which is why the divergence has stayed
invisible.

`test_changing_float` (`float_test.rb:21-33`) is blocked on the same file: it
needs `Float#changed?` (`lib/active_model/type/float.rb`, via
`Helpers::Numeric#changed?`) to answer Rails' truth table, including the
`BigDecimal("0.0") / 0` NaN arm.

Note `packages/activemodel/src/type/big-integer.test.ts` already documents the
sibling `to_i` semantics ("Rails to_i accepts leading +", "numeric string with
trailing characters extracts leading digits"), so `BigIntegerType` has the
leading-run behaviour `FloatType` is missing — worth reading as the reference
shape.

## Converged shape

Port Ruby `String#to_f` semantics for the String arm of `FloatType.cast`:
leading-run parse, `0.0` on no leading numeric run, `nil` only for blank. Keep
the existing `NaN`/`Infinity` special-case strings. Then port
`test_changing_float`'s assertions against `isChanged`.

Both `type/float_test.rb` tests currently assert trails behaviour rather than
Rails'; converge the implementation first, then the assertions — do not loosen
the Rails side.

## Acceptance criteria

- `cast("bad")` is `0`, `cast("1ignore")` is `1`, `cast("")` is `null`.
- `type/float_test.rb` reports 0 assertion-count / -kind / -value mismatches in
  `pnpm parity:test -- --assertions --package activemodel`, and the mark is
  lowered by that amount.
- AR float column round-trips still pass.
