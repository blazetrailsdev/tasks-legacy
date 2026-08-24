---
title: "parity:api:build reflows same-family receipts split by prose"
status: claimed
updated: 2026-08-24
rfc: "0106-wide-call-set-direct-burndown"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 120
priority: 2
pr: null
claim: "2026-08-24T02:09:44Z"
assignee: "api-build-reflows-same-family-tags-split-by-prose"
blocked-by: null
closed-reason: null
---

## Context

trails#6958 fixed the CROSS-family case of this: a JSDoc comment carrying both a
`@missingRailsCall` and a `@missingRailsArgs` receipt was re-flowed by either
`parity:api:build` pass while migrating nothing, because `parseJsdoc` puts the
other family's tag lines into `rest` (prose) and `renderJsdoc` re-emitted every
reconciled tag after all of it. The fix was a `slot`: `parseJsdoc`
(`scripts/api-compare/missing-rails-call-tags.ts:159,178,201`) reports the `rest`
index where THIS family's tags began, and `renderJsdoc`
(`scripts/api-compare/build.ts:262-`, the `at` local) puts the reconciled tags
back there instead of appending.

`slot` is set once — `slot ??= rest.length` at the FIRST tag of the family — and
every reconciled entry is inserted at that one index. So a comment with prose
between two receipts of the SAME family still re-flows while migrating nothing:
the second tag is hoisted up next to the first, and the blank `*` that used to
separate them is left dangling before the `*/`.

Reproduced against merged `main` (`reconcileFileText`, `--kind calls`, both calls
still flagged so neither tag is dropped):

```text
  /**                                     |   /**
   * @missingRailsCall alpha — …          |    * @missingRailsCall alpha — …
   *                                      |    * @missingRailsCall beta — …
   * Prose sitting between two receipts   |    *
   *   of the SAME family.                |    * Prose sitting between two receipts
   *                                      |    *   of the SAME family.
   * @missingRailsCall beta — …           |    *
   */                                     |    */
```

Same defect class, same blast radius as the one #6958 closed: the edit is
semantically harmless (no receipt lost, no row moved) but it is churn a reviewer
reads, and it makes `--dry-run`'s `N file(s) would change` misreport a file that
migrates nothing.

## Converged shape

Idempotency is the acceptance test, as it was for the cross-family case. Either:

- track a slot PER ENTRY — `parseJsdoc` already walks the tags in source order,
  so each `TagEntry` can carry the `rest` index it was lifted from, and
  `renderJsdoc` re-inserts each kept entry at its own index (new entries go at
  the family's first slot, keeping today's sorted-append behaviour for a mint);
  or
- keep the single slot but only take the fast path when the family's tags were
  CONTIGUOUS in the source, falling back to today's append otherwise — narrower,
  and still leaves the scattered comment re-flowing once.

The first is the real convergence; the second only shrinks the population.

Note `renderJsdoc` sorts entries by call name, so a source whose same-family tags
are not already in sorted order will still be rewritten once — that is the
existing, intended normalization and is not what this story is about.

## Acceptance criteria

- [ ] `reconcileFileText` on a comment holding two receipts of the SAME family
      with prose between them returns `text === null` in both `--kind`s, with
      both calls still flagged.
- [ ] The cross-family case #6958 fixed stays green (the existing
      `build.test.ts` "leaves a mixed-family comment alone in either kind").
- [ ] No dangling blank `*` line before `*/` in either case.
- [ ] `pnpm parity:api:calls` and `pnpm parity:api:calls:args` stay green.
