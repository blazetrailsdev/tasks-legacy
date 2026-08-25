---
title: "BigDecimal has no NAN/INFINITY form, so DecimalType returns sentinel strings from cast_value"
status: draft
updated: 2026-08-20
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activemodel"
  - "activesupport"
deps: []
deps-rfc: []
est-loc: 220
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while landing PR #6790 (move-decimal-rounding-into-activesupport-bigdecimal).

Ruby's `BigDecimal` has `BigDecimal::NAN` and `BigDecimal::INFINITY`, so
`activemodel/lib/active_model/type/decimal.rb:57-74` needs no special case for
them: `Float::NAN.to_d` is BigDecimal NAN, `Float::INFINITY.to_d` is
BigDecimal INFINITY, and `"NaN".to_d` / `"Infinity".to_d` answer the same.
Every arm of `cast_value` returns a BigDecimal.

trails' `BigDecimal`
(`packages/activesupport/src/core-ext/big-decimal/conversions.ts`) is a
sign + digit-string value with no non-finite form. `parse()` returns null for
`"NaN"` / `"Infinity"`, and the constructor throws. So
`packages/activemodel/src/type/decimal.ts` carries three sentinel-string
early returns in `castValue` and a second one in `toD`:

    if (Number.isNaN(value)) return "NaN";
    if (value === Infinity) return "Infinity";
    if (value === -Infinity) return "-Infinity";

and its return type is widened to `BigDecimal | string | null` where Rails'
is always a BigDecimal. The widening leaks outward: `DecimalType extends
ValueType<BigDecimal | string>`, `typeCastForSchema` has to branch on
`value instanceof BigDecimal`, and `applyScale` has to pass strings through
untouched. The PG adapter's `'NaN'::numeric` / `'Infinity'::numeric`
round-trip currently reads those sentinel strings.

This is the last non-Rails shape left in `decimal.ts` after #6790 removed the
seven decimal-string arithmetic helpers.

## Converged shape

Give `BigDecimal` the two non-finite forms Ruby has, then delete the
sentinels:

- A `NAN` and an `INFINITY` static (Ruby: `BigDecimal::NAN`,
  `BigDecimal::INFINITY`), with `parse()` recognising `"NaN"`, `"Infinity"`
  and `"-Infinity"` and the constructor accepting the JS `NaN` / `±Infinity`
  numbers rather than throwing.
- `isNan()` / `isInfinite()` (Ruby `BigDecimal#nan?` / `#infinite?`) and the
  propagation rules the existing arithmetic needs: `toString("F")` answers
  `"NaN"` / `"Infinity"` / `"-Infinity"`, `round` is identity on a non-finite,
  `compare` answers nil-equivalent against NAN.
- `type/decimal.ts` drops all four sentinel returns; `castValue` narrows to
  `BigDecimal | null`, and `DecimalType` narrows to `ValueType<BigDecimal>`.
- The PG bind/quote path reads `isNan()` / `isInfinite()` instead of
  comparing against the sentinel strings.

Rails source: `activemodel/lib/active_model/type/decimal.rb:57-98`.
Current deviation sites: `packages/activemodel/src/type/decimal.ts` (the
`castValue` non-finite returns, the `toD` non-finite returns, the
`BigDecimal | string` union) and
`packages/activesupport/src/core-ext/big-decimal/conversions.ts` (`parse`
returning null for a non-finite literal).

## Acceptance criteria

- [ ] `BigDecimal` carries `NAN` / `INFINITY` and answers `nan?` / `infinite?`
      at the trails spellings, with tests in
      `packages/activesupport/src/core-ext/bigdecimal.test.ts`.
- [ ] `type/decimal.ts` has no `"NaN"` / `"Infinity"` / `"-Infinity"` string
      literal; `castValue` returns `BigDecimal | null`.
- [ ] `DecimalType` is a `ValueType<BigDecimal>`; `typeCastForSchema` no
      longer branches on `instanceof BigDecimal`.
- [ ] The PG `'NaN'::numeric` / `'Infinity'::numeric` round-trip tests stay
      green reading the BigDecimal forms.
- [ ] `pnpm parity:api:extra --package activemodel` holds `type/decimal.ts` at
      <= 1 novel.
