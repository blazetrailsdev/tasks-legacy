---
title: "cache-ts-call-rows-have-no-body-bearing-declaration"
status: ready
updated: 2026-08-23
rfc: "0106-wide-call-set-direct-burndown"
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

`scripts/api-compare/call-mismatches-exclude/activesupport/cache.json` carries 4
rows (`constructor`, `keyMatcher`, `mergedOptions`, `normalizeVersion`) that
`pnpm parity:api:build --package activesupport --file cache.ts` refuses to
migrate:

    unmatched (cache.ts): constructor, keyMatcher, mergedOptions, normalizeVersion
      — no body-bearing declaration

So the migrator cannot place a `@missingRailsCall` tag for any of them, and RFC
0106 wave 5g could not clear the shard. The declarations exist in
`packages/activesupport/src/cache.ts`; the mismatch is between the artifact's
declaration key and what `build.ts` matches on (see `reconcileFileText` in
`scripts/api-compare/build.ts`).

## Acceptance criteria

- [ ] The cause of `no body-bearing declaration` for these four names is
      identified and fixed in the matcher (or the four tags written by hand, if
      the matcher is right and the declarations genuinely have no body).
- [ ] Each row is converged or leaves as a `@missingRailsCall` receipt opened
      with `PERMANENT` / `CONVERGEABLE (story …)`.
- [ ] `scripts/api-compare/call-mismatches-exclude/activesupport/cache.json` is
      deleted once empty; `pnpm parity:api:calls:tighten activesupport/cache.json`
      if the mark goes stale.
- [ ] `pnpm parity:api:calls` and `pnpm parity:api:calls:args` green.
