---
title: "Rails' array-keyed hash entry (primary_key => id) has no trails spelling"
status: draft
updated: 2026-08-02
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

Surfaced while converging `Relation#find_by_token_for` (PR 5905).

Rails writes the token lookup as a single hash entry keyed by the primary key
itself: `find_by(model.primary_key => [id])`
(`vendor/rails/activerecord/lib/active_record/token_for.rb:42`). With a
composite primary key, `model.primary_key` is an ARRAY, and Rails' predicate
builder expands an array-keyed hash entry into a composite `IN` / equality
node — one hash key covering N columns.

Trails' `findBy` / `where` has no array-hash-key form, so the ported body in
`packages/activerecord/src/relation.ts` (`findByTokenFor`) has to branch on
`Array.isArray(primaryKey)` and expand the pair list itself before calling
`findBy`. Same result, but the branch is a trails-only shape and the
deviation is repeated wherever a caller wants Rails' `primary_key => value`
spelling (`token-for.ts` carried the same expansion before PR 5905 removed
half of it).

## Acceptance criteria

- Decide whether trails' predicate builder should accept an array hash key
  (Rails' composite spelling) — read
  `vendor/rails/activerecord/lib/active_record/relation/predicate_builder.rb`
  for how Rails expands it.
- If yes: support it, then delete the `Array.isArray(primaryKey)` branch in
  `Relation#findByTokenFor` so the body is Rails' one-liner.
- If no: record why at the call site, and confirm no other ported body needs
  the same hand-expansion.
- `token_for_test.rb`'s "finds record with a composite primary key" stays
  green either way.
