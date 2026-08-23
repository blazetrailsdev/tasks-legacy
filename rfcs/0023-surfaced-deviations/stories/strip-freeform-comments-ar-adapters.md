---
title: "strip-freeform-comments-ar-adapters"
status: done
updated: 2026-08-23
rfc: "0023-surfaced-deviations"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 650
priority: null
pr: 6945
claim: "2026-08-23T20:38:54Z"
assignee: "strip-freeform-comments-ar-adapters"
blocked-by: null
closed-reason: null
---

## Context

Follow-on slice of `strip-freeform-comments-activerecord` (slice 1 swept
`packages/activerecord/src/relation/**` and landed the glob in
`eslint.config.mjs`). `blazetrails/no-freeform-comments` is registered per
glob, so the rule's `files` list extends one directory at a time and stays
green in between.

**This slice: `packages/activerecord/src/adapters/**/\*.ts`\*\* — measured ~624 comment lines / 340 blocks.

The bar (from the arel/activemodel pass and slice 1): a comment that restates
the line or branch it sits on goes, whatever its subject — including one
narrating a TypeScript deviation. What survives, survives as JSDoc carrying a
tag or a Rails citation. Rails' OWN comments are deleted too (the Ruby is
vendored and cited). A comment recording deferred work or a known-divergent
shape becomes a story, not a better comment.

## Acceptance criteria

- [ ] `packages/activerecord/src/adapters/**/*.ts` added to the `no-freeform-comments` block's `files` in
      `eslint.config.mjs`.
- [ ] `pnpm eslint --fix` applied over that glob and the deletions reviewed
      rather than taken on trust.
- [ ] `pnpm eslint` clean over the glob, and a second `--fix` run is a no-op.
- [ ] `pnpm typecheck` clean; the test files touched run green.
- [ ] Deletions that empty a block (`catch {}`) are expressed in code, not by
      restoring the comment.
- [ ] Any deferred work or known deviation found in a deleted comment is filed
      as its own story with the trails/Rails `file:line`.
