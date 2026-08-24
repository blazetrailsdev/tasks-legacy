---
title: "file-structure manifest: same PACKAGE_DIRS drift, 124 dead actionpack keys"
status: ready
updated: 2026-08-24
rfc: "0025-fidelity-verification-tooling"
cluster: null
packages: []
deps: ["rails-privates-manifest-package-dirs-drift"]
deps-rfc: []
est-loc: 60
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`scripts/build-rails-privates-manifest.ts` was not the only hand-copied copy of
the api-compare package → src dir map.
`scripts/build-rails-file-structure-manifest.ts:40-50` carries the same one,
under a comment reading:

```ts
// Mirrors scripts/build-rails-privates-manifest.ts.
const PACKAGE_DIRS: Record<string, string> = {
  ...
  actiondispatch: "packages/actionpack/src/actiondispatch",
  actioncontroller: "packages/actionpack/src/actioncontroller",
```

Same two misspellings (`action-dispatch` / `action-controller` on disk), and the
same two actionpack packages missing entirely (`abstractcontroller`,
`actionpackversion` — `vendor/sources.ts:113,118`).

Measured 2026-08-24 on a regenerated manifest: **124 of its actionpack entries
are dead keys**. Deriving the map instead takes
`eslint/rails-file-structure-method-order.json` from 1010 files / 33,170 ordered
names to **1023 / 33,391**, and its dead actionpack entries from 124 to 3 — the
3 being unported Rails files (`helpers.rb`, `rack_cache.rb`, `browser.rb`), the
same legitimate category as the privates manifest's residue.

**Why this is latent rather than red today.** `rails-file-structure-method-order`
is rolled out to arel and activemodel only (`eslint.config.mjs:447-455`), neither
of which is affected by an actionpack path. So nothing currently reads the dead
keys. That is the more dangerous failure mode, not the safer one: the Rails
source-order data for the whole of actionpack is silently absent, and would stay
absent through an enrollment that looks like a one-line `files:` change — which
is exactly how the same defect survived unnoticed in the privates manifest.

**Split from #6993 on review.** The fix was implemented and verified there
(`pnpm lint` exit 0, manifest counts above) and then removed, because the
`rails-privates-manifest-package-dirs-drift` acceptance criteria cover only
`build-rails-privates-manifest.ts` and the `rails-private-jsdoc` violations it
unlocks. This is a different manifest feeding a different lint rule and belongs
in its own PR.

## Acceptance criteria

- `build-rails-file-structure-manifest.ts` imports `PACKAGE_DIRS` from
  `scripts/api-compare/config.ts` instead of declaring its own, deleting the
  "Mirrors …" copy.
- Regenerated `eslint/rails-file-structure-method-order.json` reports 1023 files
  / 33,391 ordered names, with no dead actionpack key other than the 3 unported
  Rails files named above.
- `npx eslint packages/arel/src packages/activemodel/src` stays clean — the
  enrolled packages' dirs are unchanged by the derivation, so the newly-correct
  actionpack data must not move them.
- `pnpm lint` exits 0.
- Grep for any further copy of the map before closing: at the time of writing
  the only two were this file and the privates builder.

## Depends on

`rails-privates-manifest-package-dirs-drift` (#6993) — it introduces the
exported `PACKAGE_DIRS` / `MANIFEST_PACKAGES` in `scripts/api-compare/config.ts`
that this story imports, and the test that pins their spellings. Start after it
merges; do not branch off it.
