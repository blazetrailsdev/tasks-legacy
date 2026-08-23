---
title: "A @missingRailsCall tag on an exported top-level function does not suppress its call"
status: claimed
updated: 2026-08-23
rfc: "0106-wide-call-set-direct-burndown"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 140
priority: null
pr: null
claim: "2026-08-23T02:57:28Z"
assignee: "top-level-function-missing-rails-call-tag-does-not-suppress"
blocked-by: null
closed-reason: null
---

## Context

`scripts/api-compare/call-mismatches-exclude/activesupport/inflector.json` has
survived three consecutive RFC 0106 tail sweeps (waves 5c, 5d, 5e) untouched,
because its two rows cannot be migrated by the normal route and nobody has
diagnosed why.

Its two rows are `safe_constantize` -> `const_regexp` and
`safe_constantize` -> `match?`. On `main` those SAME two calls ALSO already
carry `@missingRailsCall` tags on `safeConstantize`'s JSDoc in
`packages/activesupport/src/inflector.ts`. So the row and the tag coexist,
which for every other shard is impossible — a tag suppresses the flag and the
row becomes stale.

Observed behaviour (wave 5c, re-confirmed in wave 5d):

- Delete the two rows -> both tags are reported STALE.
- Delete the rows AND the tags -> both calls are reported as NEW mismatches.

So the tag is not suppressing the flag here, and the row and the tag are not
interchangeable. Every other migrated shard in RFC 0106 behaves the opposite
way.

The one structural thing that distinguishes this site: `safeConstantize` is an
exported top-level `function` in `inflector.ts`, not a class member. The
suppression path is `suppressedCallsIn`
(`scripts/api-compare/missing-rails-call-tags.ts:219-230`) and the staleness
report immediately below it; RFC 0083's
`one-line-missing-rails-call-tag-silently-ignored` (done) fixed a neighbouring
parse gap in the same file, which is the likeliest shape of this one too.

Both waves reverted the shard rather than guess, so the diagnosis is still owed.

## Converged shape

Find why `suppressedCallsIn` does not associate a `@missingRailsCall` tag on an
exported top-level function's JSDoc with that function's flagged calls, and fix
the association — so that for `inflector.ts` a tag suppresses and the row is
then deletable, exactly as it is for a class member.

Then migrate the two rows the normal way
(`pnpm parity:api:build --package activesupport --file inflector.ts --call const_regexp --call 'match?'`),
prefixing each reason with a `PERMANENT` claim first, and delete the emptied
shard rather than committing `[]`.

Add a regression case under `scripts/api-compare/` covering a tag on an
exported top-level function — the existing tests only cover class members,
which is why this shipped.

## Acceptance criteria

- [ ] The reason a tag on a top-level `function` fails to suppress is
      identified and fixed in `scripts/api-compare/missing-rails-call-tags.ts`.
- [ ] A regression test covers the top-level-function tag shape and fails on
      the pre-fix code.
- [ ] `activesupport/inflector.json`'s two rows migrate to tags and the shard
      is deleted.
- [ ] `pnpm parity:api:calls` and `pnpm parity:api:calls:args` green, with no
      STALE tag and no NEW mismatch.
