---
title: "trailties has 1277 manifest-backed private names and is in neither rails-private-jsdoc glob"
status: ready
updated: 2026-08-25
rfc: "0121-internal-tag-accounting"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 250
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`trailties` is in `MANIFEST_PACKAGES`
(`scripts/api-compare/config.ts`) and the privates projection produces real data
for it — a 2026-08-25 run keys **83 files / 1277 names** under
`packages/trailties/src/` in `eslint/rails-private-methods.json`.

None of it is enforced. The `rails-private-jsdoc` block in `eslint.config.mjs`
and the `files` glob in `eslint/rails-private-jsdoc.config.mjs` both list:

```text
arel, activesupport, activemodel, actionpack, actionview, activerecord,
rack, globalid, i18n, did-you-mean
```

`packages/trailties/src/**/*.ts` is absent from both. So 1277 names that ARE
backed by Rails visibility data are neither required to carry `@internal` nor
available to the reverse rule from
`require-no-rails-equivalent-on-unbacked-internal`.

This predates `rails-privates-manifest-missing-gem-packages` (PR #7042), which
added the four gem ports to both globs but did not touch the pre-existing
trailties gap. Both configs carry a comment calling the list a "per-package
rollout; widen as packages adopt", so this is the remaining unadopted package
with manifest backing.

## Converged shape

Add `packages/trailties/src/**/*.ts` to the `rails-private-jsdoc` block in
`eslint.config.mjs` and to the `files` glob in
`eslint/rails-private-jsdoc.config.mjs` — they must stay in sync, per the header
comment on the standalone config. Then tag the members the rule reports, after
checking each against the vendored Ruby's `private`/`protected` section rather
than trusting the manifest alone.

Expect the count to be non-trivial: trailties is the largest unenrolled package
in the manifest. If the tag count makes one PR exceed the LOC ceiling, split by
subtree (`generators/`, `commands/`, …) and file the rest.

Sizing note: `packages/trailties/src/generators/app-generator.ts` is already
excluded from other lint blocks (`eslint.config.mjs`) because it holds template
strings that emit source; check whether it needs the same treatment here.

## Acceptance criteria

- [ ] `packages/trailties/src/**/*.ts` is in the `rails-private-jsdoc` block of
      `eslint.config.mjs` AND the `files` glob of
      `eslint/rails-private-jsdoc.config.mjs`.
- [ ] `npx eslint --no-inline-config -c eslint/rails-private-jsdoc.config.mjs
"packages/trailties/src/**/*.ts"` is clean.
- [ ] Every added `@internal` is verified against the vendored Ruby's
      `private`/`protected` section, not just the manifest.
- [ ] `Rails API/Test Comparison` green.
