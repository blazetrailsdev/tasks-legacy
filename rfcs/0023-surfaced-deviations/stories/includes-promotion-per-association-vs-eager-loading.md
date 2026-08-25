---
title: "includes promoted to JOIN per-association instead of Rails all-or-nothing eager_loading?"
status: ready
updated: 2026-07-27
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 200
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Rails decides eager-loading all-or-nothing: `preload_associations` (relation.rb:1321-1328)
adds `includes_values` to the preload list only `unless eager_loading?`, and
`eager_loading?` is a single boolean for the whole relation.

This port promotes `includes` to a JOIN **per association**
(`relation.ts` `_includesToPromoteFromReferences()` / `promotedIncludes`), so
`load` must subtract exactly the specs it promoted and pass the remainder
explicitly. That is why `preloadAssociations(records, assocNames?)` (PR #5331)
carries a second parameter that Rails does not have; the Rails-shaped derivation
survives only as its default value.

This is the root deviation behind that extra parameter and behind the
`includes`/`references` promotion divergences seen elsewhere.

## Acceptance criteria

- Determine whether per-association promotion is required by any passing test or
  is an unforced deviation.
- Either converge to Rails' all-or-nothing `eager_loading?` gate and drop the
  `assocNames` parameter from `preloadAssociations`, or document the deviation
  with the test(s) that pin it.
