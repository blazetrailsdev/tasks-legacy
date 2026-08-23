---
title: "Dedupe the Rails belongs_to test duplicated into a trails-only file"
status: draft
updated: 2026-08-23
rfc: "0023-surfaced-deviations"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 30
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

# Dedupe the Rails belongs_to test duplicated into a trails-only file

## Context

Surfaced classifying the 47 `associations` candidates in trails#6933 (RFC 0078
`sweep-trails-only-test-files-associations`).

`packages/activerecord/src/associations/singular-reader-stale-target.trails.test.ts`
opens `describe("BelongsToAssociationsTest")` — a Rails class name, in a file
with no Rails counterpart — and its first case is
`it("reassigning the parent id updates the object")`, a verbatim port of
`vendor/rails/activerecord/test/cases/associations/belongs_to_associations_test.rb:1252`
(`def test_reassigning_the_parent_id_updates_the_object`).

That same Rails test is ALSO ported in the convention file where
`parity:test` matches it, `associations/belongs-to-associations.test.ts:1559`.
So the copy in the trails-only file is a duplicate: it credits nothing (the
convention file already does), and it puts a Rails-named `describe` on a file
the sweep classified as a trails invention.

The file's second case,
`it("loadTarget re-queries a stale target instead of returning the cache")`, has
no Rails counterpart and is the genuine trails-only content the file exists for.

## Converged shape

The duplicated Rails case is deleted from
`singular-reader-stale-target.trails.test.ts`, leaving only the `loadTarget`
stale-target case, and the file's `describe` is renamed off the Rails class name
onto what it actually covers. The Rails test keeps its single home at
`belongs-to-associations.test.ts:1559`.

Note the CLAUDE.md constraint: test NAMES are not reworded. This story deletes a
duplicated case and retitles a trails-only `describe` that names a Rails class
it is not the counterpart of — it must not touch the surviving Rails test's name
in `belongs-to-associations.test.ts`.

## Acceptance criteria

- [ ] `singular-reader-stale-target.trails.test.ts` no longer contains the
      `reassigning the parent id updates the object` case.
- [ ] Its `describe` no longer spells `BelongsToAssociationsTest`.
- [ ] `belongs-to-associations.test.ts` is untouched and still credits the Rails
      test.
- [ ] `pnpm parity:test --package activerecord` totals are unchanged or better.
