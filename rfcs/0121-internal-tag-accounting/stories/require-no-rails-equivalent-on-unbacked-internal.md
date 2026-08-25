---
title: "Require @noRailsEquivalent alongside @internal where no Rails-private counterpart exists"
status: ready
updated: 2026-08-25
rfc: "0121-internal-tag-accounting"
cluster: null
packages:
  ["arel", "activerecord", "actionpack", "activemodel", "activesupport", "actionview", "trailties"]
deps: ["rails-privates-manifest-package-dirs-drift"]
deps-rfc: []
est-loc: 400
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

The RFC decision: `@internal` keeps its TypeDoc meaning; where an `@internal`
public member has no Rails-private counterpart it must ALSO carry
`@noRailsEquivalent`, which wins in `extract-ts-api.ts` so the member re-enters
the measured surface as `Allowed`.

Two code changes plus a burndown.

**1. Precedence in the extractor.** Today
`scripts/api-compare/extract-ts-api.ts:815,2199` (and the parallel sites at
`719`, `869`, `962`, `1990`, `2301`, `2495`) set
`internal = visibility !== "public" || hasInternalJsDocTag(decl)`, and
`scripts/api-compare/extra-surface.ts:952` drops every `internal: true` member.
A member carrying **both** tags must reach the scorer, where
`collectAllowedNames` / the `@noRailsEquivalent` path already counts it as
`Allowed` and classifies it `PERMANENT` / `CONVERGEABLE`
(`extra-surface.ts:2154-2170`). Real TS `private` / `protected` and
`#`-identifiers keep conferring `internal` unconditionally — only the JSDoc tag
yields.

**2. The reverse lint.** `eslint/rails-private-jsdoc.mjs` enforces one
direction: it _requires_ `@internal` where the Rails counterpart is private on
every host in that Ruby file, reading `eslint/rails-private-methods.json` (604
files, 10,701 names). Nothing enforces the reverse — an `@internal` on a public
member absent from the manifest is unchecked. That reverse check is this story,
over the manifest that already exists.

**3. The burndown.** 1,156 tags in the manifest-covered packages need a
`@noRailsEquivalent` reason added (874 in Rails-matched files, 282 in
trails-only files). Per package:

| package       | tags needing a receipt |
| ------------- | ---------------------- |
| activerecord  | 697                    |
| actionpack    | 229                    |
| activesupport | 65                     |
| activemodel   | 54                     |
| actionview    | 50                     |
| arel          | 40                     |
| trailties     | 21                     |

That is far past any single PR, so the lint must ship behind a per-package
enrollment set (the shape `GATED_PACKAGES` uses in
`scripts/api-compare/extra-surface.ts` for the RFC 0117 ratchet) and each
package's burndown lands as its own follow-up story. Start with `arel` (40, and
its `node-slots.ts` / `visitors/connection.ts` cases are the clearest
`PERMANENT` examples) or `trailties` (21).

**Ratchet safety.** `@noRailsEquivalent` names are removed as `allowlistedCount`
before `novelCount` / `totalExtras` are computed
(`extra-surface.ts:1702-1710`), and the mark's dimensions are `totalNovel` /
`totalExtras` (`extra-surface-mark.ts:67`). So arel's committed
`{ novel: 0, total: 63 }` (`scripts/api-compare/extra-surface-mark.json`) is
unaffected by the retag. Verify this empirically before enrolling arel — the
gate is only-shrink and there is no reseed.

Depends on `rails-privates-manifest-package-dirs-drift`: with 36% of the
manifest dead, the reverse lint would demand receipts for hundreds of actionpack
members that _do_ have a Rails-private counterpart.

## Acceptance criteria

- `extract-ts-api.ts`: a declaration carrying both `@internal` and
  `@noRailsEquivalent` is emitted with the `noRailsEquivalent` reason and
  without `internal: true`. TS `private`/`protected`/`#` are unchanged. Covered
  by a unit test per emission site touched.
- A lint rule flags a public TS declaration carrying `@internal` whose (file,
  name) is absent from `eslint/rails-private-methods.json` and which carries no
  `@noRailsEquivalent`. Message names both remedies.
- The rule is enrolled for one package only in this PR; the enrollment set is
  documented as only-grow.
- That package's tags are retagged, each with a reviewed one-line reason opening
  `PERMANENT` or `CONVERGEABLE`.
- `pnpm parity:api:extra:gate` is green and
  `scripts/api-compare/extra-surface-mark.json` is unchanged.
- Follow-up stories filed for each remaining package's burndown.
- CLAUDE.md documents the rule alongside the existing `@noRailsEquivalent` /
  `@missingRailsCall` prose.
