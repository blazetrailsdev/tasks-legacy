---
title: 'number_to_rounded never reaches Rails'' non-finite "%f" arm'
status: draft
updated: 2026-08-14
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activesupport"
deps: []
deps-rfc: []
est-loc: 90
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`NumberToRoundedConverter#convert`
(`activesupport/lib/active_support/number_helper/number_to_rounded_converter.rb:21-28`)
branches on `rounded_number.finite?` and formats a non-finite value with
`"%f" % rounded_number`:

    if rounded_number.finite?
      ...
    else
      # Infinity/NaN
      "%f" % rounded_number
    end

Rails asserts the result at `activesupport/test/number_helper_test.rb:232`:

    assert_equal "-Inf", number_helper.number_to_rounded(-Float::INFINITY, precision: 0, significant: true)

trails never reaches that arm. `NumberConverter#isValidFloat`
(`packages/activesupport/src/number-helper/number-converter.ts`) returns
`false` for a non-finite number, so `execute()` short-circuits and
`number_to_rounded(-Infinity, ...)` answers `"-Infinity"` (the raw
`String(number)`) instead of `"-Inf"`. Rails' `valid_float?` is
`Float(number, exception: false)`, which accepts an actual
`Float::INFINITY` — only an unparseable _string_ is rejected.

The finite branch is fully ported (PR #6546); this is the one arm of that
`if` that has no counterpart.

## Converged shape

- `isValidFloat` accepts a non-finite JS number (a numeric value is a valid
  Float) while still rejecting a non-numeric string.
- `convert` gains the `else` arm: when the rounded value is not finite,
  format it as Ruby's `"%f"` does — `"Inf"` / `"-Inf"` / `"NaN"` — and pass
  that through `NumberToDelimitedConverter` and `format_number` like the
  finite arm.
- `RoundingHelper#convert_to_decimal` must tolerate the non-finite value it
  is then handed (MRI's `BigDecimal("-Infinity")` is legal).

## Acceptance criteria

- [ ] Rails' `number_to_rounded(-Float::INFINITY, precision: 0, significant: true)`
      assertion (`number_helper_test.rb:232`) is live in
      `packages/activesupport/src/number-helper.test.ts` and passes as `"-Inf"`.
- [ ] Every other assertion in that file stays green, including
      `numberToRounded("x")` returning `"x"`.
