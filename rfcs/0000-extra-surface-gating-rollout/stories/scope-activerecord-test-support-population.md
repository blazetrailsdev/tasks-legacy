---
title: "Decide whether activerecord-test-support belongs in the compared population"
status: draft
updated: 2026-08-24
rfc: "0000-extra-surface-gating-rollout"
cluster: api-compare
packages: []
deps: []
deps-rfc: []
est-loc: 150
priority: 4
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`activerecord-test-support` is the fourth-largest `novel` population in the repo
and the only one whose profile suggests the MEASUREMENT is wrong rather than the
code. From the 2026-08-24 run:

| package                   | files | novel | moved | total | allowed | noCntrp |
| ------------------------- | ----- | ----- | ----- | ----- | ------- | ------- |
| activerecord-test-support | 30    | 89    | 6     | 95    | 0       | 82      |

Read the columns against each other: **89 novel out of 95 total**, only 6 moved,
**0 allowed**, and **82 no-counterpart**. Every other package has a substantial
`moved` slice — names Rails does define, in a different `.rb`. This one has
almost none, which says the comparator is finding essentially no Rails
counterpart for anything in it.

That is consistent with what the package IS: a port of Rails' _test helpers_.
Rails' `activerecord/test/` tree is not Rails public API, is not laid out like
`lib/`, and much of it (fixture accessors, connection helpers, assertion
helpers) has no `lib/` counterpart by design. If the comparator's Ruby-side
population does not include the Rails test tree, then 82 no-counterpart files
are a _mapping_ fact, not 89 acts of invention.

Burning this down before answering the scoping question could waste 89 names of
work, or worse, produce 89 `@noRailsEquivalent` tags that relabel a measurement
artifact as justified deviation — precisely what CLAUDE.md calls debt masquerading
as permission.

This story ANSWERS the question. It does not burn anything down.

## Acceptance criteria

- Determine what Ruby-side population the comparator maps
  `activerecord-test-support` against today — trace it through
  `scripts/parity/conventions.ts` (`PATH_SEGMENT_ALIASES`,
  `RUBY_FILE_TS_OVERRIDES`) and the manifest build, and name the Rails
  directories in or out.
- Sample at least 15 of the 89 novel names and classify each: (a) mirrors a real
  Rails test-helper method the comparator cannot see, (b) mirrors a real
  `lib/` method in a file we map wrongly, (c) genuinely trails-invented, (d)
  should be `@internal` (wiring, not surface). Cite `vendor/rails` `file:line`
  per name.
- Recommend ONE of: extend the Ruby-side population to cover the Rails test
  tree; add file-mapping overrides; remove the package from the compared
  population entirely; or accept it as a normal burndown target.
- If the recommendation is a burndown, file it as its own story with the sampled
  triage attached — do not start it here.
- Deliverable is a written finding (an audit doc or the story body itself), not
  a code change. Any code change beyond exploratory measurement means the scope
  was misread.
