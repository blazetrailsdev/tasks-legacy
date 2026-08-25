---
title: "numericality-exponent-number-needs-bignum-compare"
status: draft
updated: 2026-08-17
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activemodel"
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

`packages/activemodel/src/validations/numericality-validation.test.ts` carries a
`PERMANENT-SKIP`ped `it.skip("validates numericality with exponent number")`,
landed with PR #6636 (RFC 0105 assertion parity re-port).

Rails: `activemodel/test/cases/validations/numericality_validation_test.rb:311-318`

```ruby
def test_validates_numericality_with_exponent_number
  base = 10_000_000_000_000_000
  Topic.validates_numericality_of :approved, less_than_or_equal_to: base
  topic = Topic.new
  topic.approved = (base + 1).to_s
  assert_predicate topic, :invalid?
end
```

The test needs exact integer arithmetic past 2^53. Ruby Integers are
arbitrary-precision, so `(base + 1).to_s` is `"10000000000000001"` and
`"10000000000000001".to_i > 10_000_000_000_000_000`. Every JS number is a
double: `base + 1 === base`, so the test's own setup loses the digit before the
validator sees anything, and `parse_as_number`'s integer-string branch
(`numericality.ts`, mirroring numericality.rb:79-80) rounds both sides to the
same `1e16`.

Expressing it needs a BigInt-typed value through the parse pipeline and
`Comparability#compareOperator` (`packages/activemodel/src/validations/comparability.ts`),
which is shared with `ComparisonValidator` — `compareOperator` is typed
`(op, a: number, b: number)`, and `:==`/`:!=` would need loose equality to
compare a bigint against a number the way Ruby's `Integer == Float` does.
Nothing else in the validator needs that today, which is why #6636 skipped
rather than widened it.

## Acceptance criteria

- `validates numericality with exponent number` runs (no `it.skip`) and passes,
  with the body unchanged apart from whatever the value's construction needs.
- `pnpm parity:test -- --package activemodel` matched count rises by 1
  (the skip currently costs activemodel 961/963 → 960/963).
- Whatever type widening `compareOperator` takes keeps `ComparisonValidator`'s
  Temporal/string/number call sites green
  (`packages/activemodel/src/validations/comparison-validation.test.ts`).
- `pnpm parity:api:calls`, `pnpm parity:api:calls:args` and
  `pnpm parity:api:extra --package activemodel` stay clean; any deviation is
  justified at the call site with a Rails cite.
- OR: `pnpm tasks block` it with the specific blocker if a bigint-capable
  pipeline genuinely cannot be made faithful — but "it is a bigger diff" is not
  one.
