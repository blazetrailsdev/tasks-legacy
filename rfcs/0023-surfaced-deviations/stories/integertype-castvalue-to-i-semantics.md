---
title: "IntegerType#cast_value diverges from to_i rescue nil for strings and objects"
status: ready
updated: 2026-07-27
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

Surfaced by review of PR #4887 (arel-predications-unboundable-duck-types-like-rails),
which fixed the ±Infinity/NaN and raise-vs-nil halves of `cast_value` but left the
`to_i` semantics themselves diverging.

Rails' `Integer#cast_value` (`activemodel/lib/active_model/type/integer.rb:90`) is
`value.to_i rescue nil` — it calls `to_i` on the **value**. Trails
(`packages/activemodel/src/type/integer.ts:132-150`) stringifies everything and
parses:

```ts
const parsed = parseInt(String(value), 10);
return isNaN(parsed) ? null : parsed;
```

Two divergences follow:

1. **Non-numeric strings.** Ruby `"not_a_number".to_i` is **0**, so Rails'
   `cast("not_a_number")` is `0`. Trails returns `null`. Already flagged in the
   file's own comment (integer.ts:31-35) as "theoretical", and pinned by asserts
   inside `casting objects without to_i` (integer.test.ts) — a Rails test name
   whose Rails body (integer_test.rb:30-32) only asserts `Object.new`.

2. **Objects with a numeric-ish `toString`.** `String([5])` is `"5"`, so trails'
   `cast([5])` is `5`; Ruby's `Array#to_i` does not exist, so Rails is `nil`.
   Confirmed by probe on #4887. Any object whose `toString` starts with digits
   casts to a number instead of nil.

## Why it matters

Not cosmetic once RangeHandler converges
(`converge-range-handler-bind-attribute-bounds`): with bounds wrapped in
QueryAttributes, a non-numeric bound decides `serializable?` and therefore
`unboundable?`. Rails: `"abc".to_i` is 0, in range, ordinary range query. Trails:
cast is null... and once `typeCast` is also a no-op
(`query-attribute-type-cast-is-a-no-op`), `IntegerType#isSerializable("abc")`
would go through `Number("abc")` → NaN paths and can flip a normal range to
`in([])`. The three stories interlock and should be sequenced together.

## Acceptance criteria

- [ ] `castValue` mirrors `value.to_i rescue nil`: a String uses Ruby `to_i`
      semantics (leading digits, else 0 — NOT null), and a value with no `to_i`
      analogue is null rather than being stringified and parsed.
- [ ] `cast("not_a_number")` is 0, matching Rails; the assert inside
      `casting objects without to_i` that pins null is corrected (the Rails body
      only asserts `Object.new`).
- [ ] `cast([5])` / objects with numeric-prefix `toString` are null.
- [ ] `cast({})`, `Object.create(null)`, Symbol, ±Infinity, NaN stay null, and
      `cast` still never raises (integer.rb:90's rescue, kept by #4887).
- [ ] The existing documented divergence comment at integer.ts:31-35 is removed
      once it no longer describes the code.
- [ ] BigInt precision path (`narrowBigInt`) unchanged; 2^63 range tests still pass.
- [ ] parity:api / parity:test delta non-negative.
