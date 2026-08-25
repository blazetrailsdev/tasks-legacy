---
title: "number_to_human unwraps the rounded BigDecimal because BigDecimal#/ is unported"
status: draft
updated: 2026-08-14
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activesupport"
deps: []
deps-rfc: []
est-loc: 160
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`NumberToHumanConverter#convert`
(`activesupport/lib/active_support/number_helper/number_to_human_converter.rb:12-16`)
keeps the value a `BigDecimal` end to end:

    number = RoundingHelper.new(options).round(number)
    exponent = calculate_exponent(units, number.abs)
    number = number / (10**exponent)
    rounded_number = NumberToRoundedConverter.convert(number, options)

PR #6546 made `RoundingHelper#round` return a `BigDecimal` for every helper,
but `packages/activesupport/src/number-helper/number-to-human-converter.ts`
immediately unwraps it — `Number(roundedValue.toString("F"))` — because
`BigDecimal#/` is not ported in
`packages/activesupport/src/core-ext/big-decimal/conversions.ts` (which has
`mult`, `abs`, `round`, `compare`, `isZero`, `isNegative` and no division).
So `number_to_human` is the one helper still float-scaled, and a value wider
than a 53-bit mantissa loses digits before it reaches
`NumberToRoundedConverter`. The deviation is noted at the call site in that
file; this story removes the note.

## Converged shape

- Port `BigDecimal#/` (Ruby core division). Rails only ever divides by an
  exact `10**exponent` here, so an exact decimal-shift implementation covers
  the call site; Ruby's general `#/` uses `BigDecimal.limit`/precision
  digits, so decide the precision contract explicitly rather than assuming
  termination.
- In `number-to-human-converter.ts`, keep the `BigDecimal` through
  `calculateExponent(units, number.abs())` and `number / 10**exponent`,
  passing the `BigDecimal` to `NumberToRoundedConverter.convert` — which
  already accepts one.
- Delete the three-line DEVIATION comment above the unwrap.

## Acceptance criteria

- [ ] `number-to-human-converter.ts` no longer converts the rounded value to
      a JS number; the DEVIATION comment is gone, not reworded.
- [ ] `BigDecimal` division exists under its Ruby name with the precision
      contract documented.
- [ ] `number_to_human` of a value wider than a JS float (e.g.
      `"1234567890123456789012"`) keeps its digits.
- [ ] `packages/activesupport/src/number-helper.test.ts` and
      `number-helper-i18n.test.ts` stay green.
