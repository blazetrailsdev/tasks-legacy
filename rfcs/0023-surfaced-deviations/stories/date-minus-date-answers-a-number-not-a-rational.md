---
title: "Date#- against a Date answers a JS number where ruby/date answers a Rational"
status: draft
updated: 2026-08-09
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activesupport"
  - "date"
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

Shipped in PR #6286 and tracked here rather than left as prose.

`minus_without_duration` is ruby/date's own `Date#-`
(`vendor/rails/activesupport/lib/active_support/core_ext/date/calculations.rb:107`
aliases it; the implementation is `date_core.c` `d_lite_minus`). Against another
Date it answers a **Rational** day count — `Date.new(2005,2,24) - Date.new(2005,2,21)`
is `(3/1)`, not the Integer `3`.

trails' port
(`packages/activesupport/src/core-ext/date/calculations.ts`,
`minusWithoutDuration`) answers a plain JS `number`:

```ts
if (other instanceof Temporal.PlainDate) {
  return date.since(other, { largestUnit: "day" }).days;
}
```

The value is right for whole days, which is every case a `PlainDate` receiver
can produce, but the _type_ is not: Ruby callers can take `.numerator` /
`.denominator` / `.to_r` off the result, and `Date - DateTime` in Ruby is a
genuinely fractional Rational. The `date` package already ports `Rational`
(`packages/date/src/date.ts`), so the type exists to return.

Related: the same day-count reader is what
`duration_test.rb`'s `test_minus_with_duration_does_not_break_subtraction_of_date_from_date`
exercises, ported in #6286.

## Converged shape

Return the ported `Rational` from the Date-minus-Date arm, denominator 1, and
keep the numeric arm answering a `PlainDate`. Check whether
`rational-is-number-backed-not-arbitrary-precision` (0088, done) constrains the
representation before choosing the construction.

## Acceptance criteria

- `minusWithoutDuration(a, b)` for two `PlainDate`s answers a `Rational` whose
  `toI()` is the day count and whose denominator is 1.
- `minusWithDuration`'s non-Duration arm threads the same type through.
- Callers updated; the `date-ext.trails.test.ts` cover asserts the Rational,
  not a bare number.
- `pnpm parity:api` / `pnpm parity:api:calls` deltas non-negative.
