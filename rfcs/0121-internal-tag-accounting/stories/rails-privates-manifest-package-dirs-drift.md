---
title: "rails-privates manifest: PACKAGE_DIRS drift kills 36% of the manifest"
status: in-progress
updated: 2026-08-24
rfc: "0121-internal-tag-accounting"
cluster: null
packages: ["actionpack"]
deps: []
deps-rfc: []
est-loc: 120
priority: null
pr: 6993
claim: "2026-08-24T15:34:50Z"
assignee: "deanmarano"
blocked-by: null
closed-reason: null
---

## Context

`scripts/build-rails-privates-manifest.ts:49-58` hardcodes its own
`PACKAGE_DIRS` map, which has drifted from the authoritative
`PACKAGE_SRC_SUBDIR` in `scripts/api-compare/config.ts:57-63`:

| package              | manifest emitted                            | actual path on disk                            |
| -------------------- | ------------------------------------------- | ---------------------------------------------- |
| `actiondispatch`     | `packages/actionpack/src/actiondispatch/`   | `packages/actionpack/src/action-dispatch/`     |
| `actioncontroller`   | `packages/actionpack/src/actioncontroller/` | `packages/actionpack/src/action-controller/`   |
| `abstractcontroller` | _(absent from the map)_                     | `packages/actionpack/src/abstract-controller/` |
| `actionpackversion`  | _(absent from the map)_                     | `packages/actionpack/src/action-pack/`         |

Two distinct faults, not one. `actiondispatch` / `actioncontroller` were present
and misspelled. `abstractcontroller` / `actionpackversion` were never in the map
at all, so their Rails private methods have never been validated in either
direction — both are api-compared packages (`vendor/sources.ts:113,118`) whose
TS roots `packageSrcDir` already resolves via `PACKAGE_SRC_SUBDIR`. Deriving the
map only closes the gap if every package sharing the actionpack dir is in it.

`eslint/rails-private-methods.json` therefore contains **219 dead keys out of
604** — every actionpack entry. `eslint/rails-private-jsdoc.mjs` looks a file up
by `path.relative(repoRoot, filename)`, so none of them ever match and the rule
has **never fired on actionpack**, despite the standalone config
(`eslint/rails-private-jsdoc.config.mjs:24-26`) listing all four sub-packages in
its `files` glob.

Measured 2026-08-24 on a freshly regenerated manifest:

```console
npx eslint --no-inline-config -c eslint/rails-private-jsdoc.config.mjs "packages/actionpack/src/**/*.ts"
  before path fix: 0 violations
  after  path fix: 40 violations
```

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
- A test asserts every package dir the manifest projects onto exists as a
  directory. **Not** "every manifest key resolves to a file that exists" — that
  is unachievable and was wrong when this story was written: 134 keys
  legitimately name Rails files trails has not ported yet, and a dead key is
  indistinguishable from an unported one. The directory prefix is the level at
  which the drift is detectable.
- The manifest-reading half of the test skips cleanly when
  `eslint/rails-private-methods.json` is absent or empty. The file is
  **gitignored** and built only by the Ruby-bearing `rails-comparison` CI job,
  and `railsApiAvailable` writes an empty one when `rails-api.json` is missing
  (see `require-rails-api.ts` and the sibling story
  `rails-privates-manifest-silently-empty-without-api-compare-output`). There is
  consequently nothing to commit — the fix ships as code only.
- `MANIFEST_PACKAGES` covers every api-compared package mapping to the
  `actionpack` dir, pinned by a test so it cannot fall behind `PACKAGES` again.
- The actionpack violations the fix unlocks are fixed (`eslint --fix` produces
  them; review each tag lands on the right declaration).
- `npx eslint --no-inline-config -c eslint/rails-private-jsdoc.config.mjs
"packages/**/src/**/*.ts"` is clean.

### Second defect, found by the autofix

Unblocking the rule surfaced a false positive that has to be fixed here or the
PR ships wrong `@internal` tags.

The all-private guard runs on **Ruby** names, but the manifest is keyed by **TS**
name, and the two are not one-to-one: `rubyMethodToTs` gives a `?` method the
bare stem as a candidate. So the private `content_security_policy?` contributes
`contentSecurityPolicy` — the spelling of the **public**
`content_security_policy` class DSL sitting beside it in
`action_controller/metal/content_security_policy.rb:40`. `rails-private-jsdoc`
then demands `@internal` on a public Rails method and hides it from the website
API reference, which is the exact inversion of the rule's purpose.

Fix: after projecting the all-private names, subtract every candidate of every
`mixed` name in the same file, so the guard applies in the namespace it actually
gates on. Removes 66 names across 7 packages — including `permitted`,
`nestedScope`, `setContentType`, and activemodel's `attribute`, the case
`rails-private-jsdoc.mjs:10-13` cites in its own header as what the guard is for.
