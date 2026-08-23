---
title: "sweep-trails-only-test-files-associations"
status: ready
updated: 2026-08-23
rfc: "0078-sti-schema-reflection-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Follow-up slice of RFC 0078 `sweep-trails-only-test-files-onto-trails-name`,
whose first slice (`support/**`, 27 files) landed in trails#6932. That PR
regenerated the candidate list with the story's method — extract the TS test
manifest via `pnpm parity:test --package activerecord`, then take every
`packages/activerecord/src/**/*.test.ts` in
`scripts/test-compare/output/ts-tests.json` that never appears as a matched TS
counterpart in the per-file report. Result: **238** non-`.trails.` candidates,
of which **47** sit under `associations`:

```text
associations/alias-tracker.test.ts
associations/apply-association-scope.test.ts
associations/association-relation.test.ts
associations/association-scope-alias-tracker.test.ts
associations/association-scope-cache.test.ts
associations/association-scope.test.ts
associations/belongs-to-inverse-seed-composite-pk.test.ts
associations/belongs-to-stale-state-bigint-composite-fk.test.ts
associations/collection-association-bigint-number-key-match.test.ts
associations/collection-proxy-count.test.ts
associations/collection-proxy.test.ts
associations/constructor-form-and-hmt-insert.test.ts
associations/cp-count-disable-joins-through.test.ts
associations/disable-joins-association-scope.test.ts
associations/disable-joins-composite-key.test.ts
associations/disable-joins-composite-nested.test.ts
associations/disable-joins-nested-through.test.ts
associations/disable-joins-polymorphic-nonid-pk.test.ts
associations/disable-joins-routing-widening.test.ts
associations/errors.test.ts
associations/getmodelcolumns-virtual-projection.test.ts
associations/habtm.test.ts
associations/inline-polymorphic-keys.test.ts
associations/join-dependency-alias-tracker.test.ts
associations/join-dependency-belongs-to-dedup.test.ts
associations/join-dependency-duplicate-objects.test.ts
associations/join-dependency-extra-columns.test.ts
associations/join-dependency-nested-hydration.test.ts
associations/join-dependency-polish.test.ts
associations/join-dependency-quoting.test.ts
associations/join-dependency-spec.test.ts
associations/join-dependency-through-aliasing.test.ts
associations/join-dependency-walk.test.ts
associations/loader-methods.test.ts
associations/nested-through-advanced.test.ts
associations/nested-through-preloader.test.ts
associations/polymorphic-sti-through.test.ts
associations/preloader-bigint-number-key-match.test.ts
associations/preloader/through-association-hasmany-raw-join-scope.test.ts
associations/preloader/through-association-nested-join-scope.test.ts
associations/preloader/through-association-nested-raw-join-scope.test.ts
associations/preloader/through-association-raw-join-scope.test.ts
associations/reload-owner-repoint.test.ts
associations/singular-reader-stale-target.test.ts
associations/source-type-validation.test.ts
associations/sti-owner-through-foreign-key.test.ts
associations/through-association-scope.test.ts
```

The rename itself is not the substance — **classification is**. Each file is
either a trails invention (rename to `*.trails.test.ts`) or an _unported_
Rails file whose plain name is what a future port will match on (leave it, and
note the Rails file it will match). `support/**` was unambiguous because
Rails' `test/support/` holds helper sources and no `*_test.rb` at all; this
directory is not, so check each candidate against
`vendor/rails/activerecord/test/` and `pnpm rails:find` before renaming.

## Acceptance criteria

- [ ] Regenerate the candidate list (it drifts as files are ported).
- [ ] Classify every candidate under `associations` as trails invention or awaiting a
      Rails port; record the Rails counterpart for the latter.
- [ ] Rename only the trails-invention set. Do NOT touch any test name —
      renames are file-level only.
- [ ] `pnpm parity:test --package activerecord` totals byte-identical before
      and after.
- [ ] Update every hard-coded path that names a renamed file: vitest include
      globs, CI filters, `eslint/*.mjs` allowlists (see
      `eslint/test-infra-scope.mjs`), `scripts/non-transactional-row-writes.json`,
      and source-comment citations.
