---
title: "@missingRailsCall on an exported top-level function does not suppress its flag"
status: in-progress
updated: 2026-08-22
rfc: "0106-wide-call-set-direct-burndown"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 200
priority: null
pr: 6888
claim: "2026-08-22T22:41:19Z"
assignee: "missing-rails-call-tag-inert-on-top-level-function"
blocked-by: null
closed-reason: null
---

## Context

A `@missingRailsCall` tag on an **exported top-level function** does not
suppress its call-set flag, while the same tag on a class method does. Found in
`wave-5c-tail-sweep` (PR #6882), which had to revert
`activesupport/inflector.json` rather than migrate it.

`packages/activesupport/src/inflector.ts` carries, on `safeConstantize`'s JSDoc,
two tags that predate the PR:

    @missingRailsCall const_regexp — ...
    @missingRailsCall match? — ...

and `scripts/api-compare/call-mismatches-exclude/activesupport/inflector.json`
carries baseline rows for the SAME two calls
(`safe_constantize` -> `const_regexp`, `safe_constantize` -> `match?`).

Observed, on a clean tree with a fresh `pnpm build`:

- Drop the two JSON rows, keep the tags: `pnpm parity:api:calls` reports both
  tags **STALE** ("the TS body now makes the call") — so with the rows gone the
  calls are not in the flag population, which is what STALE means.
- Drop the tags too: the same gate reports both calls as **NEW mismatches**
  (`+ activesupport inflector.ts safe_constantize const_regexp`) — so the calls
  ARE in the population.

Those two results are contradictory unless the tag is failing to register as a
suppression while still being parsed well enough to be reported stale. The row,
not the tag, is doing the suppressing. That makes row and tag non-interchangeable
for this declaration, which breaks the RFC 0106 migration contract
(`pnpm parity:api:build` assumes a migrated tag suppresses exactly what its row
suppressed, and drops the row on that assumption).

`safeConstantize` is `export function safeConstantize(...)` at
`packages/activesupport/src/inflector.ts:316` — a top-level function, not a
class member. Every shard that migrated cleanly in PR #6882 was a class method
or a `this`-typed mixin function. That is the suspected discriminator, but it is
not confirmed.

Relevant code:

- `suppressedCallsIn` / `parseJsdoc` — `scripts/api-compare/missing-rails-call-tags.ts:219-233`
- `taggedCallsOn` / `taggedCommentOf` / `missingRailsCallTags` —
  `scripts/api-compare/extract-ts-api.ts:1625,1677,1714`
- `extractClass` calls `missingRailsCallTags` at `extract-ts-api.ts:2174`; the
  top-level-function extraction path is the one to compare against.

## Why this matters beyond one shard

If the non-suppression generalizes to other exported top-level functions, then
`@missingRailsCall` tags already committed on such functions are **inert**: the
gate is green only because a baseline row happens to cover the same call, and
deleting that row (a normal burndown step) will surface the call as NEW rather
than as a tag-suppressed receipt. Worth an audit of existing tags on top-level
functions once the mechanism is understood.

## Acceptance criteria

- [ ] Root-cause identified: state precisely why a `@missingRailsCall` tag on an
      exported top-level function fails to suppress, citing the extractor path.
- [ ] Fixed so a tag on a top-level function suppresses its call exactly as one
      on a class method does, with a regression test in
      `scripts/api-compare/` that FAILS on the pre-fix baseline.
- [ ] Audit: report how many committed `@missingRailsCall` tags sit on top-level
      functions and are currently inert. File follow-ups if the count is
      non-trivial.
- [ ] `activesupport/inflector.json`'s two `safe_constantize` rows
      (`const_regexp`, `match?`) migrate cleanly to their existing tags, and the
      shard is deleted.
- [ ] `pnpm parity:api:calls` and `pnpm parity:api:calls:args` green.
