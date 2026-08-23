---
title: "parity:api:build rewrites a mixed-family JSDoc comment while migrating nothing"
status: in-progress
updated: 2026-08-23
rfc: "0106-wide-call-set-direct-burndown"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 120
priority: null
pr: 6958
claim: "2026-08-23T22:26:33Z"
assignee: "api-build-reflows-the-other-tag-familys-jsdoc"
blocked-by: null
closed-reason: null
---

## Context

`parity:api:build` reconciles ONE tag family per run. A JSDoc comment carrying
tags from BOTH families is rewritten by either pass, because `parseJsdoc(comment,
origin, tag)` puts the other family's tag lines into `rest` (prose) and
`renderJsdoc` re-emits every entry AFTER the prose, under one separator.

Observed on `packages/activerecord/src/encryption/cipher/aes256-gcm.ts` `encrypt`,
whose comment holds a `@missingRailsCall order:generateIv,constructor` receipt,
then prose, then a `@missingRailsArgs generate_iv` receipt. A run of
`pnpm parity:api:build --package activerecord --kind args --file
encryption/cipher/aes256-gcm.ts` migrates 0 rows and still reports
`1 file(s) updated`, moving the prose above the args tag and leaving a doubled
blank star line:

```diff
73a74,77
>    *
>    * Ruby's `clear_text` is a byte String, whose JS pair is a Buffer, so a string
>    * argument is decoded once on entry and `clearText` is bytes from there on.
>    *
79,81d82
<    *
<    * Ruby's `clear_text` is a byte String, ...
```

The edit is semantically harmless (no receipt is lost, no row moves) but it is
churn a reviewer has to read, and it makes `--dry-run`'s `N file(s) would change`
misreport: the file is named for a rewrite that migrates nothing. The call-SET
pass has the same behaviour and it predates PR #6941 — the args pass inherited
it rather than introducing it, which is why #6941 left it alone.

## Converged shape

A reconcile pass should be a no-op on a comment whose only change is the other
family's tags being re-flowed:

- keep the other family's tag lines in place rather than folding them into
  `rest`, or
- preserve source order for tags the pass is not reconciling, and
- do not emit a second blank `*` separator when `head` already ends in one
  (`renderJsdoc` pops trailing `*` lines only while `head.length > 1`).

Idempotency is the acceptance test: reconciling a mixed-family comment twice,
in either `--kind`, must produce zero edits on the first run.

## Acceptance criteria

- [ ] `parity:api:build --kind args` on `encryption/cipher/aes256-gcm.ts`
      reports `0 file(s) would change`.
- [ ] Same for `--kind calls` on that file.
- [ ] A test in `scripts/api-compare/build.test.ts` covers a comment holding
      both a `@missingRailsCall` and a `@missingRailsArgs` receipt with prose
      between them, asserting `text === null` in both kinds.
- [ ] `pnpm parity:api:calls` and `pnpm parity:api:calls:args` stay green.
