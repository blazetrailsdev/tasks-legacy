---
title: "hash with wrong keys test asserts an unrelated behavior"
status: draft
updated: 2026-07-29
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activemodel"
deps: []
deps-rfc: []
est-loc: 30
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`packages/activemodel/src/type/date-time.test.ts` has

```ts
it("hash with wrong keys", () => {
  expect(type.cast("not-a-date")).toBe(null);
});
```

The name comes from Rails' `test_hash_with_wrong_keys`
(`vendor/rails/activemodel/test/cases/type/date_time_test.rb:34-38`), which
asserts that `type.cast(a: 1)` raises `ArgumentError` with the message
`"Provided hash {a: 1} doesn't contain necessary keys: [1, 2, 3]"`. The trails
body asserts something unrelated (an unparsable string casting to nil), so
`parity:test` matches the pair by name while the behavior Rails covers goes
untested here.

The behavior itself IS implemented and IS tested — under the invented name
`valueFromMultiparameterAssignment throws when keys 1/2/3 missing` in the same
file. So this is a naming/mapping defect, not a coverage hole. Predates
PR #5567; noticed while porting `test_string_to_time_with_timezone`.

## Acceptance criteria

- `hash with wrong keys` asserts what Rails' `test_hash_with_wrong_keys`
  asserts, including the error message.
- The duplicate invented-name test is removed or folded in, per the
  "no invented test names" rule.
- The nil-for-unparsable-string assertion is preserved somewhere appropriate
  (it maps to `test_type_cast_datetime_and_timestamp`'s `assert_nil` line).
