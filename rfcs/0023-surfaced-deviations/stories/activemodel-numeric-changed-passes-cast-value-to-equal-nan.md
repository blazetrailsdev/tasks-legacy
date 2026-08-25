---
title: "activemodel-numeric-changed-passes-cast-value-to-equal-nan"
status: draft
updated: 2026-08-12
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activemodel"
deps: []
deps-rfc: []
est-loc: 80
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced by RFC 0096 wave 2 (`naming-burndown-2-arel-activemodel`). Two
`naming` call-argument rows in
`packages/activemodel/src/type/helpers/numeric.ts` are not identifier renames.
The TS `isChanged` body inserts `normalizeBigDecimal` locals Rails does not
have (a3), **and one of them is passed to the wrong parameter** — a live
behavioral divergence.

`vendor/rails/activemodel/lib/active_model/type/helpers/numeric.rb:30-33`:

```ruby
def changed?(old_value, _new_value, new_value_before_type_cast) # :nodoc:
  (super || number_to_non_number?(old_value, new_value_before_type_cast)) &&
    !equal_nan?(old_value, new_value_before_type_cast)
end
```

Rails passes `new_value_before_type_cast` to **both** `number_to_non_number?`
and `equal_nan?`, and ignores `_new_value` entirely (the leading underscore is
Rails saying so).

trails (`packages/activemodel/src/type/helpers/numeric.ts:115-129`):

```ts
const old = normalizeBigDecimal(oldValue);
const fresh = normalizeBigDecimal(newValue);
return (
  (super.isChanged(old, fresh, newValueBeforeTypeCast) ||
    isNumberToNonNumber(old, newValueBeforeTypeCast)) &&
  !isEqualNan(old, fresh)
);
```

- `isEqualNan(old, fresh)` passes the **cast** new value where Rails passes
  `new_value_before_type_cast`. `equal_nan?` checks
  `old_value.instance_of?(new_value.class) && new_value.nan?`
  (numeric.rb:36-41), so the class comparison is against a different object in
  trails than in Rails.
- `super.isChanged(old, fresh, ...)` passes normalized values where Rails'
  `super` receives the raw `old_value` / `_new_value`.

Rows: `isChanged -> number_to_non_number?` (`RB [ref:oldValue,
ref:newValueBeforeTypeCast]` vs `TS [ref:old, ref:newValueBeforeTypeCast]`) and
`isChanged -> equal_nan?` (`RB [ref:oldValue, ref:newValueBeforeTypeCast]` vs
`TS [ref:old, ref:fresh]`).

The `normalizeBigDecimal` indirection exists because Ruby `!=` on BigDecimal is
value equality while JS `!==` is identity — that comparison belongs inside the
equality helpers (or on the BigDecimal type), not hoisted into `changed?`.

## Acceptance criteria

- [ ] `isChanged` mirrors numeric.rb:30-33: `super` and both helpers receive the
      Rails arguments, with `newValue` unused.
- [ ] The BigDecimal value-equality normalization moves to wherever the
      comparison actually happens, so `changed?` has no extra locals.
- [ ] Both `naming` rows disappear from `pnpm parity:api:calls:args:report`;
      no new `shape` rows.
- [ ] `pnpm vitest run packages/activemodel/src/type` passes, plus AR dirty /
      decimal tests. If fixing the `equal_nan?` argument changes a test
      expectation, verify the new behavior against MRI (`ruby` is on PATH)
      before changing the test.

## Absorbed: `numeric-equal-nan-takes-value-before-type-cast`

Merged in during the RFC 0023 triage pass (2026-08-18). Original title: "Numeric#changed? must pass new_value_before_type_cast to equal_nan?"

### Context

`ActiveModel::Type::Helpers::Numeric#changed?`
(`vendor/rails/activemodel/lib/active_model/type/helpers/numeric.rb:31-34`)
reads:

```ruby
def changed?(old_value, _new_value, new_value_before_type_cast)
  (super || number_to_non_number?(old_value, new_value_before_type_cast)) &&
    !equal_nan?(old_value, new_value_before_type_cast)
end
```

Rails passes `new_value_before_type_cast` to `equal_nan?`. Trails passes the
CAST new value instead, at
`packages/activemodel/src/type/helpers/numeric.ts` in `applyNumericMixin`'s
`isChanged`.

The difference is observable. Rails' `equal_nan?` also requires
`old_value.instance_of?(new_value.class)`, so writing the STRING `"NaN"` over a
Float NaN attribute IS a change in Rails (String is not Float, and
`Float::NAN != Float::NAN` makes `super` true). Under trails' cast-value
variant it is NOT a change.

The deviation was surfaced and documented during #5374 but deliberately left
alone: converging it flips six existing assertions that currently pin the
cast-value behaviour.

- `packages/activemodel/src/type/float.test.ts` -
  `isChanged returns false for NaN-to-NaN when raw is "NaN" string - equal_nan? uses cast value`
- `packages/activemodel/src/dirty.test.ts` - five cases in
  `numeric type.isChanged integration via dirty tracking` and
  `clearAttributeChanges clears forced-dirty state`, each doing
  `m.writeAttribute("ratio", "NaN")` after a Float NaN cast.

Those trails-invented test names encode the deviation rather than Rails
behaviour, so converging means rewriting them against
`vendor/rails/activemodel/test/cases/type/` and
`vendor/rails/activemodel/test/cases/dirty_test.rb` rather than preserving them.

### Acceptance criteria

- `applyNumericMixin`'s `isChanged` passes `newValueBeforeTypeCast` to
  `isEqualNan`, matching numeric.rb:31-34.
- The six pinning assertions are re-derived from the Rails tests, not merely
  inverted; test names that describe the deviation are replaced with the Rails
  names they should have had.
- Decimal's `"NaN"` sentinel handling (added in #5374) keeps working: the
  sentinel stands in for BigDecimal NaN and mixed representations still count
  as changed.
- `pnpm parity:api:calls` baseline does not grow; parity:test delta is
  non-negative.
