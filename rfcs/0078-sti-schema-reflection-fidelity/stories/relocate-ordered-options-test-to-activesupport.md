---
title: "relocate-ordered-options-test-to-activesupport"
status: in-progress
updated: 2026-08-24
rfc: "0078-sti-schema-reflection-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: 6970
claim: "2026-08-24T03:21:39Z"
assignee: "relocate-ordered-options-test-to-activesupport"
blocked-by: null
closed-reason: null
---

## Context

`packages/activerecord/src/ordered-options.test.ts` is a port of ActiveSupport's
`OrderedOptionsTest` — it opens `describe("OrderedOptionsTest")` and imports
`OrderedOptions` / `InheritableOptions` from `@blazetrails/activesupport`. It
touches no ActiveRecord surface at all, and `packages/activesupport/src/ordered-options.test.ts`
already exists next to `packages/activesupport/src/ordered-options.ts`.

It surfaced during the RFC 0078 sweep (`sweep-trails-only-test-files-remaining`,
trails#TBD) as one of 81 activerecord test files with no matched Rails
counterpart. It was deliberately left alone there: renaming it to
`*.trails.test.ts` would mislabel a Rails port as a trails invention, and
deleting or moving it is not a rename.

Rails' own file is `activesupport/test/ordered_options_test.rb` (not vendored
under `vendor/rails/activesupport/test/` in this checkout — confirm against a
fresh `pnpm vendor:fetch` before deciding what the activesupport file should
contain).

## Acceptance criteria

- [ ] Diff `packages/activerecord/src/ordered-options.test.ts` against
      `packages/activesupport/src/ordered-options.test.ts`; fold any case the
      activesupport file is missing into it, verbatim (no test-name changes).
- [ ] Delete `packages/activerecord/src/ordered-options.test.ts`.
- [ ] `pnpm parity:test` totals for both packages non-negative.
- [ ] Update any hard-coded path naming the deleted file (`eslint/*` allowlists,
      `vitest.config.ts`, `scripts/*.json`).
