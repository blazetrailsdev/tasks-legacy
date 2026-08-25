---
title: "strip-freeform-comments-ar-remaining-dirs"
status: draft
updated: 2026-08-21
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 700
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Final follow-on slice of `strip-freeform-comments-activerecord`. Slice 1 swept
`packages/activerecord/src/relation/**`; sibling stories cover `(root)`,
`connection-adapters`, `associations`, `support` and `adapters`. This story
covers everything left, measured comment lines / blocks:

    372/157 type-virtualization · 323/168 encryption · 233/168 test-helpers
    115/58 sqlite · 83/42 tasks · 61/38 attribute-methods · 61/24 test-fixtures
    54/33 validations · 51/31 scoping · 35/16 database-configurations
    34/27 migration · 25/17 type · 10/3 cases · 6/1 persistence · 6/3 testing
    5/1 coders · 4/3 trailties · 2/2 locale · 1/1 middleware

Ship them in whatever grouping fits under the LOC ceiling — the rule's `files`
list is the gate, so it extends one glob at a time and stays green in between.
Once every directory is enrolled, collapse the list to
`packages/activerecord/src/**/*.ts`.

The bar (from the arel/activemodel pass and slice 1): a comment that restates
the line or branch it sits on goes, whatever its subject. What survives,
survives as JSDoc carrying a tag or a Rails citation. Rails' OWN comments are
deleted too. A comment recording deferred work becomes a story.

## Acceptance criteria

- [ ] Each remaining `packages/activerecord/src/<dir>/**/*.ts` glob added to the
      `no-freeform-comments` block's `files` in `eslint.config.mjs`.
- [ ] `pnpm eslint --fix` applied and the deletions reviewed rather than taken
      on trust; a second `--fix` run is a no-op.
- [ ] `pnpm typecheck` clean; the test files touched run green.
- [ ] Any deferred work or known deviation found in a deleted comment is filed
      as its own story with the trails/Rails `file:line`.
