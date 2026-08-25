---
title: "Relation#offset should accept a string like Rails"
status: ready
updated: 2026-07-27
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 40
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Rails' `test_finder_with_offset_string`
(vendor/rails/activerecord/test/cases/finder_test.rb:1754) is
`assert_nothing_raised { Topic.offset("3").to_a }` — `offset` accepts a
string. trails types `Relation#offset` as taking a `number`, so the ported
test in `packages/activerecord/src/finder.test.ts` (merged in #5274) has to
write `CanonicalTopic.offset("3" as unknown as number)` with a call-site
comment. The runtime path works; only the type signature diverges.

## Acceptance criteria

- `Relation#offset` (and the `offset` value in `relation/query-methods.ts`)
  accepts `number | string`, matching Rails' `offset(value)`.
- The `as unknown as number` cast and its justification comment in
  `finder.test.ts` "finder with offset string" are removed.
- Coverage that a string offset produces the same SQL as the numeric one.
