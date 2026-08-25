---
title: "Type::Decimal has no ::Numeric (Rational) cast arm"
status: ready
updated: 2026-08-25
rfc: "0113-branch-and-guard-parity"
cluster: invented-arm
packages: []
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

`vendor/rails/activemodel/lib/active_model/type/decimal.rb:60-78`:

    def cast_value(value)
      casted_value = \
        case value
        when ::Float
          convert_float_to_big_decimal(value)
        when ::Numeric
          BigDecimal(value, precision || BIGDECIMAL_PRECISION)
        when ::String
          begin value.to_d rescue ArgumentError then BigDecimal(0) end
        else
          if value.respond_to?(:to_d) then value.to_d else cast_value(value.to_s) end
        end
      apply_scale(casted_value)
    end

`packages/activemodel/src/type/decimal.ts`'s `_castWithoutScale` ports the
Float, String, BigDecimal and bigint arms but has NO `::Numeric` arm and no
`respond_to?(:to_d)` fallback — a Rational lands in the trailing
`return null`. `Rational` is a real trails type
(`packages/date/src/date.ts`, exported from `@blazetrails/date`, which
activemodel already depends on).

Surfaced by RFC 0105's activemodel assertion-parity work (PR #6639): three
tests in `vendor/rails/activemodel/test/cases/type/decimal_test.rb` assert on
Rational input and cannot be ported faithfully until the arm exists —
`test_type_cast_decimal_from_rational_with_precision` (precision: 2 →
`BigDecimal("0.33")` / `BigDecimal("0.67")`),
`…_with_precision_and_scale` (precision: 4, scale: 2), and
`…_without_precision_defaults_to_18_36` (→ `BigDecimal("0.333333333333333333E0")`,
i.e. the `BIGDECIMAL_PRECISION = 18` default at decimal.rb:47). Our port
currently substitutes invented float inputs and `toBeCloseTo` assertions for
all three.

## Converged shape

- Add the `::Numeric` arm to `_castWithoutScale`, spelled as Rails spells it:
  `BigDecimal(value, precision || BIGDECIMAL_PRECISION)` with
  `BIGDECIMAL_PRECISION = 18` as a named constant (decimal.rb:47), reached by a
  Rational (and any future non-Float numeric) rather than by falling through to
  `null`.
- Add the `respond_to?(:to_d)` else-arm, or record why the duck-typed fallback
  has no TS analogue at the call site.
- Then port the three decimal_test.rb tests verbatim, replacing the invented
  float/`toBeCloseTo` stand-ins.

## Acceptance criteria

- `type/decimal_test.rb` reports 0 assertion-count / -kind / -value mismatches
  in `pnpm parity:test -- --assertions --package activemodel`.
- `scripts/test-compare/assertion-mismatch-mark.json` lowered accordingly.
- `pnpm parity:api:calls` / `:args` green with no new rows.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0113-branch-and-guard-parity`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
