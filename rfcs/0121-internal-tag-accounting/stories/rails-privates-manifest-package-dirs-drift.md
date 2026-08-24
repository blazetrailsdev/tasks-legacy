---
title: "rails-privates manifest: PACKAGE_DIRS drift kills 36% of the manifest"
status: ready
updated: 2026-08-24
rfc: "0121-internal-tag-accounting"
cluster: null
packages: ["actionpack"]
deps: []
deps-rfc: []
est-loc: 120
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`scripts/build-rails-privates-manifest.ts:49-58` hardcodes its own
`PACKAGE_DIRS` map, which has drifted from the authoritative
`PACKAGE_SRC_SUBDIR` in `scripts/api-compare/config.ts:57-63`:

| manifest emits                                | actual path on disk                            |
| --------------------------------------------- | ---------------------------------------------- |
| `packages/actionpack/src/actiondispatch/`     | `packages/actionpack/src/action-dispatch/`     |
| `packages/actionpack/src/actioncontroller/`   | `packages/actionpack/src/action-controller/`   |
| `packages/actionpack/src/abstractcontroller/` | `packages/actionpack/src/abstract-controller/` |
| `packages/actionpack/src/actionpackversion/`  | `packages/actionpack/src/action-pack/`         |

`eslint/rails-private-methods.json` therefore contains **219 dead keys out of
604** — every actionpack entry. `eslint/rails-private-jsdoc.mjs` looks a file up
by `path.relative(repoRoot, filename)`, so none of them ever match and the rule
has **never fired on actionpack**, despite the standalone config
(`eslint/rails-private-jsdoc.config.mjs:24-26`) listing all four sub-packages in
its `files` glob.

Measured 2026-08-24 on a freshly regenerated manifest:

````console
npx eslint --no-inline-config -c eslint/rails-private-jsdoc.config.mjs "packages/actionpack/src/**/*.ts"
  before path fix: 0 violations
  after  path fix: 40 violations
```console

All 40 are the rule's own autofixable "add an `@internal` tag" message
(`optimizeRoutesGeneration`, `contentSecurityPolicy`, `handleConditionalGetBang`,
…). actionpack already carries 829 `@internal` tags of which 375 match the
corrected manifest, so this is a small residue, not a wall.

The remaining 134 dead keys are a **separate** gap — packages `PACKAGE_DIRS`
omits entirely — and are tracked by
`rails-privates-manifest-missing-gem-packages`.

## Acceptance criteria

- `build-rails-privates-manifest.ts` derives its TS directory per package from
  `scripts/api-compare/config.ts` (`packageSrcDir` / `PACKAGE_DIR_OVERRIDES` +
  `PACKAGE_SRC_SUBDIR`) rather than a private copy, so the two cannot drift
  again.
- A test asserts every key in the regenerated manifest resolves to a file that
  exists on disk (guarded so it skips cleanly when `rails-api.json` is absent —
  the manifest is only buildable in the Ruby-bearing CI job, see
  `require-rails-api.ts` and the sibling story
  `rails-privates-manifest-silently-empty-without-api-compare-output`).
- `eslint/rails-private-methods.json` is regenerated and committed.
- The 40 actionpack violations are fixed (`eslint --fix` produces them; review
  each tag lands on the right declaration).
- `npx eslint --no-inline-config -c eslint/rails-private-jsdoc.config.mjs
  "packages/actionpack/src/**/*.ts"` is clean.
````
