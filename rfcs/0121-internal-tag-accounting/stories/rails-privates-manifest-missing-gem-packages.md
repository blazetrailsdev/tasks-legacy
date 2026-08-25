---
title: "rails-privates manifest covers no gem packages: 381 unvalidatable @internal tags"
status: done
updated: 2026-08-25
rfc: "0121-internal-tag-accounting"
cluster: null
packages: []
deps: ["rails-privates-manifest-package-dirs-drift"]
deps-rfc: []
est-loc: 200
priority: null
pr: 7042
claim: "2026-08-25T15:38:33Z"
assignee: "pg-table-definition-takes-unlogged-as-an-option-rails-reads-the-adapter"
blocked-by: null
closed-reason: null
---

## Context

`scripts/build-rails-privates-manifest.ts:49-58` maps only eight api-compare
packages to TS directories. The vendored-Ruby extractor
(`scripts/api-compare/extract-ruby-api.rb`) covers more than that — a 2026-08-24
run reports `rack: 75 classes … (79 internal)`, `globalid: 12 classes …
(30 internal)`, `i18n: 30 classes … (85 internal)`, `did-you-mean: 12 classes …
(10 internal)` — and their TS ports carry `@internal` tags that no manifest can
validate:

| package          | `@internal` tags | of which TS private/protected |
| ---------------- | ---------------- | ----------------------------- |
| date             | 287              | 0                             |
| rack             | 82               | 37                            |
| globalid         | 38               | 17                            |
| i18n             | 11               | 0                             |
| activerecord-cli | 6                | 0                             |
| html-sanitizer   | 6                | 0                             |
| did-you-mean     | 5                | 0                             |

**381 public-member tags with no Rails-privates data behind them.** Both
directions of enforcement are blind here: `rails-private-jsdoc` cannot require
the tag, and the reverse rule from
`require-no-rails-equivalent-on-unbacked-internal` cannot distinguish "no Rails
counterpart" from "manifest does not cover this package", so it would demand a
`@noRailsEquivalent` receipt for all 381 including the correctly-tagged ones.

`date`, `html-sanitizer` and `activerecord-cli` are a separate case again — they
are not api-compare packages at all (`vendor/sources.ts` / `apiComparePackages()`),
so there may be no Ruby side to project from. Establish that per package before
assuming the fix is uniform.

## Acceptance criteria

- Determine, per package above, whether a Rails/gem source exists in
  `vendor/` that the privates projection can run over, and record the answer in
  the story or RFC.
- For those that do: add them to the manifest's package→dir map (derived, per
  `rails-privates-manifest-package-dirs-drift`) and regenerate; report the
  resulting matched/unmatched split.
- For those that do not: document them as permanently outside the manifest, and
  make the reverse lint's enrollment set unable to include them (so it cannot
  demand receipts it has no basis for).
- `eslint/rails-private-jsdoc.config.mjs`'s `files` glob and the
  `rails-private-jsdoc` block in `eslint.config.mjs` stay in sync with whatever
  set lands.
