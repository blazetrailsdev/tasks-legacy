---
title: "Count transaction control statements in the query-assertion helpers"
status: draft
updated: 2026-07-27
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
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

Surfaced in #5449 (port-hollow-has-many-replace-tests).

Rails' `assert_no_queries` counts transaction control statements, so
`test_replace_with_same_content`
(`vendor/rails/activerecord/test/cases/associations/has_many_associations_test.rb:2087`)
is a real guard on `CollectionAssociation#replace`'s same-record-list short
circuit: without the short circuit the second `firm.clients = []` opens a
transaction and the `BEGIN`/`COMMIT` trip the assertion.

Our `assertNoQueries` / `assertQueriesCount`
(`packages/activerecord/src/testing/query-assertions.ts:61,93`) do not count
the statements emitted by `_replaceTransaction`
(`packages/activerecord/src/associations/collection-proxy.ts:3466`). Verified
empirically on #5449: deleting the `sameRecordList` short circuit at
`collection-proxy.ts:3386` leaves the ported `replace with same content` test
passing. The port is faithful to Rails but is currently a no-op guard, and
every other ported `assert_no_queries` around a transaction-wrapped write has
the same blind spot.

This is distinct from `restore-dropped-assert-queries-count-in-ported-ar-tests`
(that story restores assertions that were dropped entirely; this one is about
assertions that are present but blind).

## Acceptance criteria

- [ ] Determine whether `BEGIN` / `COMMIT` / `SAVEPOINT` / `RELEASE SAVEPOINT`
      reach the query subscriber the assertion helpers count, and why they are
      currently excluded.
- [ ] Make the helpers count transaction control statements the way Rails'
      `assert_queries_count` does.
- [ ] Deleting the `sameRecordList` short circuit at `collection-proxy.ts:3386`
      must make `replace with same content` FAIL.
- [ ] Fix or triage any ported test whose query counts shift once transaction
      statements are counted.
