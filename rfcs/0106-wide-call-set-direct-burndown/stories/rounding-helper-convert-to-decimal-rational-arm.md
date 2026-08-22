---
title: "rounding-helper convert_to_decimal drops the Rational arm and its digit_count call"
status: done
updated: 2026-08-22
rfc: "0106-wide-call-set-direct-burndown"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 60
priority: null
pr: 6881
claim: "2026-08-22T20:35:01Z"
assignee: "parity-api-build-must-not-drop-harvested-tags"
blocked-by: null
closed-reason: null
---

## Context

`packages/activesupport/src/number-helper/rounding-helper.ts`
`convertToDecimal` drops the `Rational` arm of
`vendor/rails/activesupport/lib/active_support/number_helper/rounding_helper.rb:25-34`:

```ruby
def convert_to_decimal(number)
  case number
  when Float, String
    BigDecimal(number.to_s)
  when Rational
    BigDecimal(number, digit_count(number.to_i) + options[:precision])
  else
    number.to_d
  end
end
```

`digit_count` (`rounding_helper.rb:20-23`) is called from TWO sites — the
`Rational` arm above, and `absolute_precision` (:36-42). The port makes the
`absolute_precision` call but not the `Rational` one, because trails has no
`Rational`, so the whole `when Rational` branch has no value to match.

This is the last remaining row in
`scripts/api-compare/call-mismatches-exclude/activesupport/number-helper/rounding-helper.json`
after #6869 migrated the file's `fetch` row to a `@missingRailsCall` tag. Its
reason already reads "Convergeable with the Rational port" — it was left in the
baseline rather than tagged precisely because it is NOT a permanent language
shortcoming, so it must not be closed by writing a `PERMANENT` justification.

Depends on a trails `Rational`. `packages/date` already carries Rational-adjacent
machinery (see the `Rational(n,1)` canonicalization work), so scope this against
whatever `Rational` seat that package settles on rather than inventing a second.

## Acceptance criteria

- [ ] `convertToDecimal` has the `Rational` arm, in Rails' branch order
      (`Float`/`String`, then `Rational`, then the `to_d` else), calling
      `digitCount(number.toI())` with `options.precision` added, per
      `rounding_helper.rb:29-30`.
- [ ] The `digit_count` row is DELETED from
      `scripts/api-compare/call-mismatches-exclude/activesupport/number-helper/rounding-helper.json`
      (the shard is then empty and the file removed), not migrated to a tag and
      not reworded. If the row is the only one left, `pnpm parity:api:calls:tighten
activesupport/number-helper/rounding-helper.json` clears the resulting stale mark.
- [ ] `pnpm parity:api:calls`, `parity:api:calls:args`, `parity:api:reasons`,
      `parity:api:detached` green.
