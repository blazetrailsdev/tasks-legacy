---
title: "sweep-trails-only-test-files-relation"
status: claimed
updated: 2026-08-24
rfc: "0078-sti-schema-reflection-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: null
claim: "2026-08-24T02:45:38Z"
assignee: "sweep-trails-only-test-files-relation"
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
of which **15** sit under `relation`:

```text
relation-exec-main-query.test.ts
relation/arel-ast-convergence.test.ts
relation/bound-sql-literal-relation.test.ts
relation/build-arel-helpers.test.ts
relation/build-joins-from-subquery-dedup.test.ts
relation/composite-where.test.ts
relation/eager-shared-alias-tracker.test.ts
relation/finder-methods.test.ts
relation/left-outer-joins-values-structural-dedup.test.ts
relation/predicate-builder/association-query-value.test.ts
relation/query-attribute.test.ts
relation/ruby-inspect.test.ts
relation/select-star-join-collision.test.ts
relation/thenable.test.ts
relation/unscope-coverage.test.ts
relation/value-accessor-semantics.test.ts
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
- [ ] Classify every candidate under `relation` as trails invention or awaiting a
      Rails port; record the Rails counterpart for the latter.
- [ ] Rename only the trails-invention set. Do NOT touch any test name —
      renames are file-level only.
- [ ] `pnpm parity:test --package activerecord` totals byte-identical before
      and after.
- [ ] Update every hard-coded path that names a renamed file: vitest include
      globs, CI filters, `eslint/*.mjs` allowlists (see
      `eslint/test-infra-scope.mjs`), `scripts/non-transactional-row-writes.json`,
      and source-comment citations.
