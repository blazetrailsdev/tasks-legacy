---
title: "Audit and restore assert_queries_count assertions dropped from ported AR tests"
status: ready
updated: 2026-07-27
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 250
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

PR #5319 (story `belongs-to-required-validates-fk-false`) found that the trails
ports of three Rails tests in
`packages/activerecord/src/associations/belongs-to-associations.test.ts`
(`runs parent presence check if parent changed or nil`, `skips parent presence
check if parent has not changed`, `runs parent presence check if parent has not
changed and belongs_to_required_validates_foreign_key is set`) had silently
dropped the `assert_queries_count` assertions that are the ENTIRE point of those
tests in Rails
(`vendor/rails/activerecord/test/cases/associations/belongs_to_associations_test.rb:1745-1794`
asserts 4 / 3 / 4). Without them the tests passed under either value of
`belongsToRequiredValidatesForeignKey` — they asserted only that
`ship.name === "Leviathan"`. #5319 restored those four wraps using
`assertQueriesCount` (`packages/activerecord/src/testing/query-assertions.ts:61`).

This is very likely not an isolated port slip: a Rails test whose only assertion
is `assert_queries_count` becomes a no-op test when the count assertion is
dropped, and `parity:test` still counts it as ported because the test NAME
matches. That is a fidelity hole the existing tooling cannot see.

## Acceptance criteria

- Enumerate Rails AR tests that use `assert_queries_count` /
  `assert_no_queries` / `assert_queries_match` and whose trails counterpart
  (matched by test name) lacks a corresponding `assertQueriesCount` /
  `assertNoQueries` / query-matching assertion. Ship the inventory.
- Restore the missing assertions where the port already exists and the helper
  is available, keeping Rails' exact expected counts. Do NOT rename tests.
- Where a restored count does not hold in trails, do not weaken the assertion —
  file the divergence as its own story with the Rails file:line and the
  observed trails count.
- If the inventory is larger than one PR, ship the inventory plus the first
  batch and register the remainder as follow-up stories.
