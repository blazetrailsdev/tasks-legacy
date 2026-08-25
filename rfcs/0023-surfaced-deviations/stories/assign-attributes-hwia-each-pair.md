---
title: "assign-attributes-hwia-each-pair"
status: ready
updated: 2026-07-04
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activesupport"
deps: []
deps-rfc: []
est-loc: null
priority: 9000000
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Follow-up surfaced in PR #4529 (assign-attributes-each-pair-duck-typing).

Rails' `ActiveModel::AttributeAssignment#assign_attributes` admits any object
responding to `each_pair` (attribute_assignment.rb:29-30). `HashWithIndifferentAccess`
is a `Hash` subclass, so it inherits `Hash#each_pair` and is a legal argument to
`model.assign_attributes(new HashWithIndifferentAccess({...}))`.

trails HWIA (activesupport/src/hash-with-indifferent-access.ts:14) stores entries
in a private `Map`, so it fails BOTH sides of trails' assignment path:

1. the `assertHashAttributes` guard (attribute-assignment.ts) — HWIA's prototype
   isn't `Object.prototype` and it exposes neither `permitted` nor `toH`, so the
   tightened guard now raises `ArgumentError`.
2. the downstream `_assignAttributes` `Object.entries(attrs)` loop
   (attribute-assignment.ts:38) — can't see HWIA's private Map, so even pre-PR it
   silently no-op'd rather than assigning.

This was always broken (silent no-op before, ArgumentError after #4529); the guard
PR did not regress a working path. Real support requires teaching both the guard
(admit an `each`/`toHash`/`entries` duck-type) AND the assignment loop (normalize
HWIA to a plain record before `Object.entries`).

## Acceptance criteria

- [ ] `model.assignAttributes(new HashWithIndifferentAccess({name: "x"}))` assigns
      `name` instead of raising or silently no-op'ing.
- [ ] `assertHashAttributes` admits HWIA (and any object duck-typing its each-pair
      iteration) without admitting Date/Time/Map/Set.
- [ ] `_assignAttributes` iterates HWIA's entries (normalize via `toHash()`).
- [ ] Test asserts round-trip assignment from an HWIA argument.
