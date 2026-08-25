---
title: "finder.test 'find with string' ports findBySql smoke, not Rails' string-id cast assertion"
status: ready
updated: 2026-07-27
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 20
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`packages/activerecord/src/finder.test.ts:768` — `it("find with string")` —
does not port Rails' `test_find_with_string`
(vendor/rails/activerecord/test/cases/finder_test.rb:186), which asserts
`Topic.find(1).title == Topic.find("1").title` (string id casts to the same
record through the bind — the exact behavior PR #5139 converged). Our test
instead declares a bespoke inline `Topic` with `attribute("title")`, creates a
row, and asserts `findBySql('SELECT * FROM "topics"')` returns an array —
neither the string-id find nor the canonical Topic model/fixtures.

## Acceptance criteria

- [ ] `it("find with string")` asserts
      `(await Topic.find(1)).title === (await Topic.find("1")).title` using
      the canonical `test-helpers/models/topic.ts` + `fixtures(["topics"])`
      (no bespoke inline model).
- [ ] Test name unchanged (parity:test match).
