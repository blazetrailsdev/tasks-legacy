---
title: "Split load_async_test.rb whole-file exclusion: enroll portable surface cases"
status: draft
updated: 2026-08-13
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

From PR #6466's exclusion audit: `relation/load_async_test.rb` (38 Rails
tests) is fully excluded via the `future_result.rb` row in
`scripts/parity/unported-files/baseline.json` ("Thread-pool scheduled query…
Test file fully excluded — all live test classes exercise
FutureResult/scheduled? semantics"). The mechanism (Concurrent thread pool,
blocking #value) genuinely doesn't port, but the public surface —
`Relation#load_async` returning a relation whose records resolve
asynchronously (activerecord/lib/active_record/relation.rb, `load_async`) —
maps naturally onto trails' Promise-backed relations, and a subset of the 38
tests assert surface behavior (records loaded, calling twice, scoping) rather
than scheduler internals.

Converged shape: audit the 38 tests in
`vendor/rails/activerecord/test/cases/relation/load_async_test.rb`; split the
whole-file exclusion into case-level `tests:` exclusions for the
scheduler-internal cases and enroll the portable surface cases against a
`loadAsync` port.

## Acceptance criteria

- A written disposition per test case (portable vs scheduler-internal), in the
  registry as case-level exclusions with reasons.
- Portable cases enrolled (stubs acceptable; they count as pending, not
  matched).
