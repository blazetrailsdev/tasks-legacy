---
title: "CONVERGEABLE @noRailsEquivalent reasons must carry a story id"
status: draft
updated: 2026-08-24
rfc: "0000-extra-surface-gating-rollout"
cluster: api-compare
packages: []
deps: []
deps-rfc: []
est-loc: 250
priority: 2
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

A `@noRailsEquivalent` reason opens with `PERMANENT` or `CONVERGEABLE` (RFC
0080's permanence claim; `scripts/api-compare/lint-extra-surface-ratchet.ts:70`
restates the grammar in its error text). `CONVERGEABLE` asserts a promise —
"this will be removed" — and today nothing records who will remove it or when.

The live failure: the Ruby `Tempfile` block-form receipts at
`packages/activerecord/src/encryption/encrypted-file.ts:144` and
`packages/activerecord/src/tasks/postgresql-database-tasks.ts:213` were both
classed **PERMANENT** when the truth was "nobody wrote the shared helper yet".
PERMANENT was the only label that did not lie about _tracking_, because
CONVERGEABLE with no story to point at is an untracked promise. Story
`port-ruby-tempfile-block-form` now exists, and those two tags should point at
it.

Current classification by grep over `packages/*/src`: 220 PERMANENT, 15
CONVERGEABLE, 17 where the keyword wraps onto a JSDoc continuation line.

## Acceptance criteria

- Grammar: `@noRailsEquivalent CONVERGEABLE <story-slug> — <reason>`. The slug
  is the bare story id (no RFC prefix), matching the tasks-repo convention.
- The extractor validates the slug SHAPE only (kebab-case, non-empty) and errors
  with a `file:line` the way the empty-reason check already does. Existence is
  NOT checked here — that needs the tasks checkout and belongs in the periodic
  re-audit.
- `PERMANENT` is unchanged and takes no slug.
- The 17 continuation-line cases are resolved: either they classify correctly
  once the parser flattens continuation lines (the `parseJsdoc` rule
  `missing-rails-call-tags.ts` uses), or they are real unclassified tags and get
  a classification. Say which in the PR body.
- Every existing `CONVERGEABLE` tag gains a story id, or is reclassified. The
  two Tempfile receipts point at `port-ruby-tempfile-block-form` and flip
  PERMANENT → CONVERGEABLE.
- Enforcement is an ERROR (the run already fails on empty reasons and stale
  tags, so this joins an existing failure mode rather than adding a gate).

## Notes

Deliberately NOT part of this story: making open CONVERGEABLE tags block
enrollment. The RFC argues the inversion — blocking on them makes PERMANENT the
attractive misclassification, which is exactly the failure above. CONVERGEABLE
is ratcheted, not gated.
