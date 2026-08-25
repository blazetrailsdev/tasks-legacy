---
title: "ComparisonValidator's private compare() has no Rails counterpart (public_send off the value)"
status: draft
updated: 2026-08-17
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activemodel"
  - "date"
deps: []
deps-rfc: []
est-loc: 200
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced by PR #6626, which extended
`packages/activemodel/src/validations/comparison.ts`'s `private compare()`.

Rails has no such method. `ComparisonValidator#validate_each`
(`vendor/rails/activemodel/lib/active_model/validations/comparison.rb:27`)
dispatches the comparison off the value itself:

```ruby
unless value.public_send(COMPARE_CHECKS[option], option_value)
```

trails instead calls a private `compare(a, b)` returning a `<=>`-style number
and feeds it to `compareOperator` (`comparability.ts`, which carries its own
`@noRailsEquivalent PERMANENT` for the missing `public_send`). `compare()` is a
hand-written dispatch table over `Temporal.*` classes, `number`, `string`,
`Date` and (since #6626) any object exposing `compareTo`. `NumericalityValidator`
(`numericality.rb:60`) has the same `public_send` shape and its own copy of the
problem.

The `compareTo` arm is the closest thing to the Rails shape that exists: it is
trails' spelling of Ruby's `<=>` (`packages/date/src/date.ts:5147`, where
`DateInfinity` derives the `Comparable` operators from it).

## Converged shape

Give the comparable value types a `compareTo` (Ruby `<=>`) and let
`validate_each` dispatch through it plus `compareOperator` alone, so the
per-type `if` ladder disappears and one Rails method stays one TS method. The
`Temporal.*` classes are third-party and cannot carry `compareTo`, so the
residual ladder should shrink to a single spaceship helper shared with whatever
`Comparable` seat the port settles on — not a private method on the validator.

Decide at claim time whether the shared seat belongs in `@blazetrails/date` (next
to `DateInfinity#compareTo`) or in activesupport, and whether
`NumericalityValidator` converges in the same pass.

## Acceptance criteria

- [ ] `ComparisonValidator` has no private `compare`; the comparison goes
      through one `<=>`-shaped seat and `compareOperator`.
- [ ] `packages/activemodel/src/validations/comparison-validation.test.ts`
      stays at 0 assertion-count/kind/value mismatches.
