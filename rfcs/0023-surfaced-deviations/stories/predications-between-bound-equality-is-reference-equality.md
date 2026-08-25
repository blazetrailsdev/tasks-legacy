---
title: "between's degenerate-range arm uses JS === where Ruby == is value equality"
status: draft
updated: 2026-08-22
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "arel"
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

Surfaced while folding `predications-range.ts` into `predications.ts` (#6856).

`Arel::Predications#between` (`activerecord/lib/arel/predications.rb:56`) has the
degenerate-range arm:

```ruby
elsif other.begin == other.end
  eq(other.begin)
```

Ruby `==` on the bounds is **value** equality: two equal `Date`s, `Time`s,
`BigDecimal`s or `Arel::Nodes::Casted`s compare equal
(`Casted#==` is `nodes/casted.rb`'s `eql?`/`==` pair).

`packages/arel/src/predications.ts` `between` ports it as
`other.begin === other.end` — JS reference equality. Identical primitives still
collapse to `eq`, but two equal `Date` objects (the common
`between(someDate, sameDate)` shape from AR's RangeHandler) do NOT, and the
predicate builds a `Between(And(Casted, Casted))` where Rails builds an
`Equality`. Same divergence for `notBetween`'s bounds, which reach the same
comparison indirectly.

The `===` predates #6856 (it came over verbatim from the extracted
`betweenFromRange`); that PR reproduced it rather than widening scope.

## Converged shape

Compare the bounds the way Ruby's `==` does for the value classes an Arel bound
can hold. Check whether trails already has a Ruby-`==` analogue to reuse before
adding one (`Casted#eql`, `Node#eql`, activesupport comparisons); a bespoke
deep-equal helper in `predications.ts` would be new invented surface and is not
the answer.

Related idiom class: RFC 0082 (`0082-ruby-ts-idiom-conversion-classes`) tracks
Ruby-vs-JS equality/truthiness conversion classes.

## Acceptance criteria

- `between({ begin: d, end: d2 })` with two equal-valued `Date` bounds builds
  `Nodes::Equality`, matching `predications.rb:56-57`.
- A regression test that FAILS on the current `===` (see
  `packages/arel/src/predications-range.test.ts`'s "range begin == end collapses
  to Equality" case, which only covers primitives today).
- `pnpm vitest run packages/arel` green; AR range/where suites green.
