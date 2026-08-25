---
title: "validate_each reports the raw value to filtered_options and duplicates EachValidator's allow_blank arm"
status: draft
updated: 2026-08-13
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activemodel"
deps: []
deps-rfc: []
est-loc: 120
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced by PR #6457, which converged `validate_each`'s call ORDER (dropping a
trails-only nil pre-arm and moving `parse_as_number` below the `only_integer`
check). Two operand divergences in the same body were left in place.

`activemodel/lib/active_model/validations/numericality.rb:36-47`:

```ruby
def validate_each(record, attr_name, value, precision: Float::DIG, scale: nil)
  unless is_number?(value, precision, scale) ... end
  if allow_only_integer?(record) && !is_integer?(value) ... end

  value = parse_as_number(value, precision, scale)   # <- REASSIGNS value
```

Rails **reassigns `value`**, so every `filtered_options(value)` in the reserved-
options loop below (`:48-64`) reports the PARSED number. That value reaches i18n
as the `value:` interpolation (`filtered_options`, `:113-117`).
`packages/activemodel/src/validations/numericality.ts` binds a separate local
(`const num = parseAsNumber(Number(value), precision, scale)`) and leaves `value`
holding the RAW input, so `filteredOptions(value)` and the `withCount` closure
both report the raw string where Rails reports the number. Visible in the
`:greater_than` / `:in` / `:odd` / `:even` message payloads.

Second, trails' `validateEach` carries its own
`if (this.options.allowBlank && isBlank(value)) return;` arm. Rails'
NumericalityValidator has no such line — `allow_blank` short-circuits once in
`EachValidator#validate` (`activemodel/lib/active_model/validator.rb:150-157`):

```ruby
next if (value.nil? && options[:allow_nil]) || (value.blank? && options[:allow_blank])
value = prepare_value_for_validation(value, record, attribute)
```

and it tests the **cast** value, BEFORE `prepare_value_for_validation` swaps in
the raw `*_before_type_cast` input. The trails arm tests the PREPARED value, so
the two disagree exactly where the cast and the raw form differ in blankness —
the split `test_allow_nil_works_for_casted_value` depends on for the nil arm.

## Converged shape

1. Reassign: `value = parseAsNumber(value, precision, scale)` at
   `numericality.rb:47`'s position, and let the branches below read `value`, so
   `filteredOptions(value)` reports the parsed number as Rails does. (Note the
   port also passes `Number(value)` rather than `value` into `parseAsNumber` —
   converge that argument at the same time.)
2. Delete the `allowBlank` arm from `validateEach` and confirm
   `EachValidator#validate`'s `allow_blank` short-circuit is wired and tests the
   cast value, not the prepared one.

Related: [[numericality-validate-each-reserved-options-loop]] rewrites the loop
these operands feed; landing that first makes (1) a one-line change.

## Acceptance criteria

1. `validateEach` reassigns `value` from `parseAsNumber` at Rails' line, and
   every `filteredOptions(value)` below reports the parsed value.
2. The trails-only `allowBlank` arm is gone from `validateEach`, with
   `allow_blank` handled once in `EachValidator#validate` against the cast value.
3. Rails' numericality tests stay green, including
   `test_allow_nil_works_for_casted_value`.
4. `pnpm parity:api:calls` and `pnpm parity:api:calls:args` green.
