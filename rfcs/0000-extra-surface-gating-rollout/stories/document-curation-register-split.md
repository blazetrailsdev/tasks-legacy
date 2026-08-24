---
title: "Document the per-site vs aggregate curation split"
status: ready
updated: 2026-08-24
rfc: "0000-extra-surface-gating-rollout"
cluster: api-compare
packages: []
deps: []
deps-rfc: []
est-loc: 120
priority: 3
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

RFC 0080 moved 27 method-level justifications out of
`scripts/api-compare/extra-surface-allow.json` into `@noRailsEquivalent`, and
closed. Since then the question "should register X move to JSDoc too?" has no
written answer, so it gets re-derived — and re-argued — every time someone looks
at the `scripts/` trees.

The distinction that must not be blurred: a **per-site exclusion** is a fact
about one declaration and belongs in JSDoc at that declaration; a **mark /
high-water number** is an aggregate over a package and has nowhere in JSDoc to
live. There is also a THIRD bucket the two-way split does not cover — registers
whose subject is not a TS declaration at all.

The current inventory, which this story records:

- **Aggregates, stay JSON:** `extra-surface-mark.json`,
  `test-compare/assertion-mismatch-mark.json`, `call-mismatches-unreviewed/**`
  (8 shards).
- **Not TS declarations, stay JSON:** `schema-compare/invented-baseline.json`
  (table names), `parity/unported-files/baseline.json` (patterns over _Ruby_
  file paths).
- **Per-site, migrating:** `call-mismatches-exclude/**` — under the
  review-then-migrate contract, because
  `missing-rails-call-tags.ts:20-46` makes seed prose non-suppressing and RFC
  0083 forbids minting a tag that carries it.
- **Already drained, deleted:** `arity-exclude.json`, `inheritance-exclude.json`,
  `body-pins.json` (story `retire-drained-api-compare-registers`).
- **Per-site but out of family:** `eslint/*-exclude.json` — ESLint's native
  inline form is `eslint-disable-next-line` with a description, not a JSDoc tag.
  Named so it is not forgotten; not chartered here.

## Acceptance criteria

- A doc under `docs/infrastructure/` (beside the existing
  `api-build-stub-generation-plan.md`, which RFC 0080 named as the home of the
  tag convention) records the four-bucket classification above, with each
  register named and bucketed.
- It states the review-then-migrate contract for `call-mismatches-exclude` and
  cites `missing-rails-call-tags.ts:20-46` as the reason bulk conversion is
  impossible — so the next person does not propose it.
- It states the rule for future registers: an aggregate stays JSON; a per-site
  fact about a TS declaration gets a tag in the existing family; anything whose
  subject is not a TS declaration stays JSON regardless of granularity.
- CLAUDE.md's register list links to it.
- Docs-only, so exempt from the LOC ceiling — but keep it a reference table, not
  an essay.
