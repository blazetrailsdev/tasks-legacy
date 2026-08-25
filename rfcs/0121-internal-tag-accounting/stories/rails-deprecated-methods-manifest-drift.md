---
title: "rails-deprecated-methods-manifest-drift"
status: ready
updated: 2026-08-25
rfc: "0121-internal-tag-accounting"
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

`eslint/rails-deprecated-methods.json` is committed, but regenerating it with
`pnpm rails-privates:manifest` (which `pnpm parity:api` also runs) produces a
STRICTLY SMALLER manifest than the committed one:

```diff
-    "packages/activerecord/src/connection-adapters/mysql/schema-definitions.ts": [
-      "_unsignedDecimal", "_unsignedFloat", "unsignedDecimal", "unsignedFloat"
-    ],
-    "packages/activerecord/src/connection-handling.ts": ["_connection", "connection"],
-    "packages/activesupport/src/core-ext/benchmark.ts": ["_ms", "ms"]
+    "packages/activerecord/src/connection-handling.ts": ["connection"]
```

Both states claim "3 files (8 names)" / the same run banner, so the drift is
silent. It was noticed only because the regenerated file was swept into an
unrelated PR (#7045) by `git add -A` and a reviewer caught the diff.

At least one committed row is provably dead:
`packages/activesupport/src/core-ext/benchmark.ts` **does not exist** — only
`benchmark.test.ts` does. So the committed manifest is carrying rows for a file
that was deleted or moved, which means `blazetrails/rails-deprecated-jsdoc` has
been silently enforcing nothing for that entry.

The other two rows need a decision rather than a rubber stamp — the generator
dropping `_unsignedDecimal`/`unsignedDecimal` and narrowing `connection-handling`
from `["_connection", "connection"]` to `["connection"]` may be correct (the TS
side changed) or may be the SAME `PACKAGE_DIRS`-style drift that killed 36% of
`rails-private-methods.json` in `rails-privates-manifest-package-dirs-drift`.
Find out which before regenerating: a manifest that only ever shrinks is how a
lint rule stops enforcing anything.

Relevant code: `scripts/build-rails-privates-manifest.ts` (`emitDeprecatedManifest`,
and `diffDeprecatedManifest` in `scripts/deprecated-manifest-diff.ts`, which
already has a `--check-deprecated` CI mode meant to catch exactly this).

## Acceptance criteria

- Root cause named for each of the three dropped/narrowed rows: dead file,
  legitimate TS-side change, or generator drift.
- `eslint/rails-deprecated-methods.json` matches what the generator produces on
  a clean checkout, with any genuine loss of coverage repaired at the generator
  rather than by hand-editing the manifest.
- The `--check-deprecated` guard actually fails on this drift (it evidently did
  not), or the reason it cannot is written down.
- `pnpm vitest run eslint/rails-deprecated-jsdoc.test.mjs` green.
