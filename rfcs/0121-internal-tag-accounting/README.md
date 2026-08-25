---
rfc: "0121-internal-tag-accounting"
title: "Constrain @internal to Rails-private surface; account for the rest"
status: draft
created: 2026-08-24
updated: 2026-08-25
owner: "@deanmarano"
packages:
  - "activerecord"
  - "actionpack"
  - "activemodel"
  - "activesupport"
  - "actionview"
  - "arel"
  - "trailties"
clusters: []
related-rfcs:
  - "0025-fidelity-verification-tooling"
  - "0080-api-compare-jsdoc-metadata"
priority: 3
---

## Problem

`@internal` is not merely documentation. It removes a name from the
`parity:api:extra` population:

- `scripts/api-compare/extract-ts-api.ts:815,2199` set
  `internal = visibility !== "public" || hasInternalJsDocTag(decl)`.
- `scripts/api-compare/extra-surface.ts:952` — `walkTsFileSurface` does
  `if (m.internal === true) return;` and drops the name entirely.

So tagging a public TS member `@internal` makes it vanish from the extra-surface
count, with no accounting of any kind. Compare `@noRailsEquivalent`, the
_tracked_ escape hatch (RFC 0080): allowed extras are counted in the `Allowed`
column, classified `PERMANENT` / `CONVERGEABLE`, and gated for staleness and for
being unclassified (`extra-surface.ts:2154-2170`).

`@internal` is an **untracked exit from the measured surface**, and it is the
larger of the two by an order of magnitude.

## Measurement (2026-08-24)

Method: regenerate `scripts/api-compare/output/rails-api.json` (the vendored-Ruby
extractor) and `eslint/rails-private-methods.json`, then walk every line-leading
`@internal` JSDoc tag in `packages/*/src/**/*.ts` (excluding `*.test.ts`) with
the TS compiler and cross each (file, name) against the manifest. A second
projection, built by flipping `build-rails-privates-manifest.ts`'s `all-private`
filter to `mixed`, gives the "Rails counterpart exists but is public" column.

**4,267 `@internal` tags.** With the path drift of story 1 corrected:

| package                                                                          | total    | Rails-private ✓ | Rails-public ⚠ | TS private/protected | no Rails counterpart |
| -------------------------------------------------------------------------------- | -------- | --------------- | -------------- | -------------------- | -------------------- |
| activerecord                                                                     | 2340     | 1324            | 145            | 174                  | 697                  |
| actionpack                                                                       | 829      | 375             | 173            | 52                   | 229                  |
| activemodel                                                                      | 257      | 183             | 14             | 6                    | 54                   |
| activesupport                                                                    | 168      | 68              | 16             | 19                   | 65                   |
| actionview                                                                       | 134      | 42              | 31             | 11                   | 50                   |
| arel                                                                             | 68       | 17              | 11             | 0                    | 40                   |
| trailties                                                                        | 36       | 6               | 8              | 1                    | 21                   |
| **covered total**                                                                | **3832** | **2015 (53%)**  | **398**        | **263**              | **1156**             |
| date / rack / globalid / i18n / did-you-mean / activerecord-cli / html-sanitizer | 435      | —               | —              | 54                   | 381                  |

53% of `@internal` tags are backed by a Rails private/protected counterpart.
**1,156 are not**, and **874 of those sit in a Rails-matched file** (282 are in
trails-only files).

The "Rails-public" column is deliberately soft: the `mixed` projection is
per-file, so a name public on any host in that Ruby file counts. It over-reports
and needs per-host verification before any of it is called a fidelity bug.

### What the unmatched tags actually are

Not mostly slot setters. Sampling the 874:

- `activerecord/src/relation/finder-methods.ts` (35): `performFind`,
  `performFirst`, `performTake`, `performSecondToLastBang`, `performFortyTwo`, …
  — a trails-invented async-dispatch layer, one `perform*` per Rails finder,
  with no Rails counterpart at all.
- `activerecord/src/associations.ts` (30): `guardCanonicalNameShadow`,
  `canonicalModelAutoloadIndex`, `_wireInverseAssociation`,
  `_cacheSingularTarget`, `resolveAssocClass`, …
- `activerecord/src/base.ts` (18): `_associationCacheStore`,
  `_reinstateConstructorDirtiness`, `_extractAssociationAttrs`,
  `ensureSchemaLoaded`, …

This is exactly the invented surface `parity:api:extra` exists to count. The
genuinely defensible cases — arel's `node-slots.ts` zero-import slot module (the
CLAUDE.md-sanctioned pattern, 17 names), `arel/src/visitors/connection.ts`'s
host interface (11 method signatures), `test-helpers/` — are real but a
minority.

## The TypeDoc collision, and why the literal rule does not work

`eslint/rails-private-jsdoc.mjs:6-8` records a second, legitimate purpose: the
website's TypeDoc build runs with `excludeInternal: true`
(`packages/website/typedoc.config.mjs:24,34`), so the tag keeps Rails-private
surface out of the generated API reference.

TypeDoc's `excludeTags` (`typedoc.config.mjs:26`) strips the _tag_ from rendered
output; it does **not** exclude the member. Only `excludeInternal` hides a
member. So simply swapping `@internal` → `@noRailsEquivalent` on the 1,156
would publish ~1,150 trails-internal names into the public API reference. A
third doc-only tag has the same problem plus a new TypeDoc option to plumb.

## Decision

> **`@internal` keeps its TypeDoc meaning and gains no new one. Where an
> `@internal` public member has no Rails-private counterpart, it must ALSO carry
> `@noRailsEquivalent`, and `@noRailsEquivalent` wins in `extract-ts-api.ts` —
> the member re-enters the measured surface as `Allowed`.**

Every `@internal` is then either backed by a Rails private/protected tag or
backed by a reviewed, `PERMANENT`/`CONVERGEABLE`-classified receipt. The
untracked exit closes.

### Why this shape and not the alternatives

- **Zero doc regression.** `excludeInternal` still hides the member.
- **Zero ratchet movement.** `@noRailsEquivalent` names are pulled out as
  `allowlistedCount` _before_ `novelCount` / `totalExtras` are computed
  (`extra-surface.ts:1702-1710`), and the mark's two dimensions are
  `totalNovel` / `totalExtras` (`extra-surface-mark.ts:67`). A tagged member
  costs the mark nothing.
- **Reuses existing machinery.** `eslint/rails-private-jsdoc.mjs` already
  enforces one direction over `eslint/rails-private-methods.json` (604 files,
  10,701 names). This RFC is that rule's reverse direction plus a precedence
  change.

The rejected alternative — making `extract-ts-api.ts` stop honoring `@internal`
outright and trusting only real TS `private`/`protected` plus the manifest — is
sharper but breaks `arel`'s ratchet on day one. `walkTsFileSurface` is reached
for **uncovered** TS files too (`extra-surface.ts:1593-1600`: `rubyFile: null`
→ empty allowed set → every name is `novel`), so arel's `node-slots.ts`,
`temporal-tag.ts` and `test-helpers/connection.ts` all re-enter as novel against
a committed mark of `novel: 0, total: 63`
(`scripts/api-compare/extra-surface-mark.json`). It also re-admits ~1,500 names
repo-wide with no receipts, which is the opposite of the goal.

## Scope

1. `rails-privates-manifest-package-dirs-drift` — the manifest path bug that
   makes 36% of the enforcement dead. Prerequisite: without it the measurement
   above cannot be reproduced and actionpack cannot be enrolled.
2. `require-no-rails-equivalent-on-unbacked-internal` — the rule: reverse-
   direction lint plus the `extract-ts-api.ts` precedence change, behind
   per-package enrollment so the ~1,156-tag burndown lands package by package.
3. `rails-privates-manifest-missing-gem-packages` — the 381 tags in `date`,
   `rack`, `globalid`, `i18n`, `did-you-mean`, `html-sanitizer` and
   `activerecord-cli` that no manifest covers at all.

## Out of scope

- The 398 "Rails counterpart is public" tags. That is a **visibility-parity**
  question, already owned by RFC 0025's `add-visibility-parity-gate` and
  `score-private-surface-in-parity-api-extra`. This RFC does not touch it, and
  the soft per-file projection above is not evidence enough to act on.
- Removing any invented surface. A `@noRailsEquivalent` tag is a receipt, not
  absolution; converging `performFind` and friends is separate work.
