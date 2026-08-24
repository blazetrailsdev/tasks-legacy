---
title: "strip-freeform-comments-ar-connection-adapters-abstract"
status: in-progress
updated: 2026-08-24
rfc: "0119-connection-adapter-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: 6985
claim: "2026-08-24T14:09:23Z"
assignee: "strip-freeform-comments-ar-connection-adapters-abstract"
blocked-by: null
closed-reason: null
---

## Context

Slice 3 of `strip-freeform-comments-ar-connection-adapters`. Slice 1 swept
`packages/activerecord/src/relation/**`; slice 2 (this story's parent) swept
`packages/activerecord/src/connection-adapters/{mysql,mysql2,sqlite3,postgresql}/**/*.ts`
and left the sub-glob in `eslint.config.mjs`'s `no-freeform-comments` block.

Remaining here: **`packages/activerecord/src/connection-adapters/abstract/**/\*.ts`**
— measured 145 rule findings (`pnpm exec eslint <glob> --rule
'{"blazetrails/no-freeform-comments":["warn",{"report":true}]}'`). Heaviest
files: `abstract/connection-pool.ts`(25),`abstract/database-statements.ts`(25),`abstract/temporal-wire.ts`(9),`abstract/schema-statements.ts` (8).

The bar: a comment that restates the line or branch it sits on goes, whatever
its subject — including one narrating a TypeScript deviation. What survives,
survives as JSDoc carrying a tag or a Rails citation. Rails' OWN comments are
deleted too. A comment recording deferred work becomes a story, not a better
comment.

## Acceptance criteria

- [ ] `packages/activerecord/src/connection-adapters/abstract/**/*.ts` added to
      the `no-freeform-comments` block's `files` in `eslint.config.mjs`.
- [ ] `pnpm eslint --fix` applied over that glob and the deletions reviewed.
- [ ] `pnpm eslint` clean over the glob; a second `--fix` run is a no-op.
- [ ] `pnpm typecheck` clean; the test files touched run green.
- [ ] Deletions that empty a block are expressed in code, not by restoring the
      comment.
- [ ] Deferred work found in a deleted comment is filed as its own story.
