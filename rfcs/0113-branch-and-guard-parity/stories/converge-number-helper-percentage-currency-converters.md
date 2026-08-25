---
title: "converge-number-helper-percentage-currency-converters"
status: ready
updated: 2026-08-25
rfc: "0113-branch-and-guard-parity"
cluster: guard-parity
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

Surfaced by RFC 0096 story `naming-burndown-activesupport` (criterion 4: rows
that are structural, not identifier renames, are left in place and filed).

Two number-helper converters diverge from Rails in body structure, which is
what the call-argument comparator reports as a `naming` row:

- `number-to-percentage-converter.ts:11-12` destructures `format`, `locale`,
  and `negativeFormat` out of the options before calling
  `NumberToRoundedConverter.convert(this.number, roundedOptions)`. Rails passes
  the whole `options` hash — `number_to_percentage_converter.rb:11`:
  `rounded_number = NumberToRoundedConverter.convert(number, options)`.
- `number-to-currency-converter.ts:21-40` reshapes Rails'
  `number_to_currency_converter.rb:10-25` entirely: it has no `valid_bigdecimal`
  arm, computes `Number(this.number)` / `Math.abs` instead of Rails'
  `number_d` / `number_d.abs`, and picks `negative_format` from a plain
  `isNegative` boolean rather than Rails'
  `format = options[:negative_format] if (number_d * 10**options[:precision]) >= 0.5`
  rounding-aware guard.

The currency one is a behavioural divergence, not only a naming one: Rails
keeps the `-` format only when the value still rounds away from zero.

## Acceptance criteria

1. `NumberToPercentageConverter#convert` passes `options` to
   `NumberToRoundedConverter.convert`, matching
   `number_to_percentage_converter.rb:11`.
2. `NumberToCurrencyConverter#convert` mirrors
   `number_to_currency_converter.rb:10-25` line for line, including the
   `valid_bigdecimal` arm, the `number_d`/`number_s` locals, and the
   `(number_d * 10**options[:precision]) >= 0.5` guard.
3. The three `naming` rows for these two files clear in
   `pnpm parity:api:calls:args:report`.
4. `pnpm vitest run packages/activesupport/src/number-helper` green, plus any
   Rails-mirrored number-helper tests that the corrected rounding guard now
   makes passable.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0113-branch-and-guard-parity`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
