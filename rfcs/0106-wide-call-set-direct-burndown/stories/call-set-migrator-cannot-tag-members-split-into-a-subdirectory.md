---
title: "parity:api:build cannot tag a member trails split out of its Rails file"
status: done
updated: 2026-08-23
rfc: "0106-wide-call-set-direct-burndown"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 160
priority: null
pr: 6904
claim: "2026-08-23T10:42:30Z"
assignee: "call-set-migrator-cannot-tag-members-split-into-a-subdirectory"
blocked-by: null
closed-reason: null
---

## Context

Four reviewed `kind: "set"` rows in
`scripts/api-compare/call-mismatches-exclude/activesupport/cache.json` cannot
leave the baseline by the sanctioned route. `pnpm parity:api:build --package
activesupport --file cache.ts` reports:

    unmatched (cache.ts): constructor, keyMatcher, mergedOptions, normalizeVersion
      — no body-bearing declaration

The rows are:

    activesupport cache.ts initialize        delete
    activesupport cache.ts key_matcher       call
    activesupport cache.ts merged_options    merge
    activesupport cache.ts normalize_version try

This is NOT the declaration-shape problem
`call-set-migrator-skips-non-body-bearing-declarations` fixed (a `this`-typed
mixin function, an arrow-function property, an object-literal method). All four
targets are ordinary `MethodDeclaration`s. The problem is **file attribution**:

- Rails declares them inside `class Store` in ONE file —
  `activesupport/lib/active_support/cache.rb:188` (`class Store`), with
  `key_matcher` at `cache.rb:779`, `merged_options` at `cache.rb:861` and
  `normalize_version` at `cache.rb:990`.
- trails split that class out, so the declarations live in
  `packages/activesupport/src/cache/store.ts` — `mergedOptions` at line 680,
  `keyMatcher` at 778, `normalizeVersion` at 830, and the `constructor` at 276.
- The baseline rows are keyed `tsFile: "cache.ts"`, which is where the
  `cache.rb` → TS path convention points, so `build.ts` searches `cache.ts`,
  finds no such declarations, and refuses all four.

Any Ruby file whose class trails splits into a subdirectory module has the same
shape, so the fix is general, not a cache special case.

## Converged shape

`build.ts` resolves a row's tag target by the declaration the compare manifest
actually matched the pair in, rather than by the row's `tsFile` path alone — so
a member matched in `cache/store.ts` is tagged there. The row key can stay as
it is; only the lookup changes.

Verify by migrating all four rows above with
`pnpm parity:api:build --package activesupport --file cache.ts` (or whatever
spelling the fix settles on) and confirming `pnpm parity:api:calls` stays green
with the shard deleted.

## Acceptance criteria

- [ ] `parity:api:build` places a `@missingRailsCall` tag on a member whose TS
      declaration lives in a different file from the one the row's `tsFile`
      names, without a per-file special case.
- [ ] The four `activesupport/cache.json` rows migrate and the shard is deleted,
      not committed as `[]`.
- [ ] `pnpm parity:api:calls` and `pnpm parity:api:calls:args` green.
