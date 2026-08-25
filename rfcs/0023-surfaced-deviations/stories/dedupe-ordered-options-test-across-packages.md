---
title: "Delete the activerecord copy of OrderedOptionsTest, keeping one owned by activesupport"
status: draft
updated: 2026-08-16
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
  - "activesupport"
deps: []
deps-rfc: []
est-loc: 150
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced in #6611: the SQLite lane went red on a test file I had not run, because
`OrderedOptionsTest` exists TWICE in trails:

- `packages/activesupport/src/ordered-options.test.ts`
- `packages/activerecord/src/ordered-options.test.ts` (imports from
  `@blazetrails/activesupport`, header says "Tests to increase Rails test coverage
  matching")

Rails has one file, `activesupport/test/ordered_options_test.rb`, and `parity:test` maps
a trails test to it by name — so the duplicate contributes nothing to coverage while
doubling the blast radius of any `OrderedOptions` change. It cost a full CI round here:
the activerecord copy still asserted the pre-convergence trails behaviour (own-only
`to_h`/`each`/`inspect`, a `?` arm `method_missing` never had, `has`) and only failed in
CI after the activesupport copy was green locally.

Neither copy is a superset: the activerecord one has `introspection` via `in`, the bang
arms and `key`; the activesupport one has `dig`, `isExtractableOptions` and the
`isOverridden` cases.

## Update (2026-08-18, after PR #6692)

Most of this is now settled, and the "union" shape below is **superseded** — do not
merge the two files.

PR #6692 rewrote `packages/activesupport/src/ordered-options.test.ts` as a real
line-by-line port of all 27 `def test_*` in `ordered_options_test.rb` (it had been
invented: Rails test names over made-up bodies). It reports 0 assertion-count, 0
assertion-value and 0 extra; the only residue is 2 assertion-kind rows from the
Symbol/String `key?` collapse, commented at the call sites.

So the activesupport copy is no longer "missing" anything the activerecord copy has —
the activerecord copy's extra assertions are invented, not coverage. Folding them in
would re-import exactly the made-up behaviour #6692 removed.

Also settled: `parity:test -- --package activerecord` credits nothing for this file
(`ordered_options_test.rb` is an activesupport Rails file), so deleting it cannot drop
the activerecord percent. Verified in #6692.

Also note: three of the activerecord copy's assertions encoded pre-convergence bugs in
`OrderedOptions` that #6692 fixed (`to_s` returning the `#<...>` form where Rails
inherits `Hash#to_s`; `in` treated as a membership test where `respond_to_missing?`
answers true for every name). They were patched in place there rather than deleted,
because deleting the 260-line file would have blown that PR's LOC ceiling.

## Converged shape

Delete `packages/activerecord/src/ordered-options.test.ts`. Nothing to migrate —
`packages/activesupport/src/ordered-options.test.ts` is already the complete port.
Confirm `pnpm parity:test` is non-negative for both packages afterwards.

## Acceptance criteria

- [ ] `packages/activerecord/src/ordered-options.test.ts` is deleted; every assertion it
      made that the activesupport copy lacked has moved there under the same test name.
- [ ] `pnpm parity:test` delta is non-negative.
- [ ] No other AR-side test file duplicates an activesupport-owned Rails test file (grep
      for the same `describe` name across packages while you are here; file separately if
      there are more).
