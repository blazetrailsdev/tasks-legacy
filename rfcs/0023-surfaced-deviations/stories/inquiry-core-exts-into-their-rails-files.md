---
title: "Move String#inquiry / Array#inquiry into their core_ext files, retiring the arrayInquiry alias"
status: draft
updated: 2026-08-17
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activesupport"
deps: ["array-utils-splits-into-rails-core-ext-array-files"]
deps-rfc: []
est-loc: 140
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Rails puts the two `inquiry` core exts in their own core_ext files, separate
from the inquirer classes they construct:

- `vendor/rails/activesupport/lib/active_support/core_ext/string/inquiry.rb:11-13`
  — `class String; def inquiry; ActiveSupport::StringInquirer.new(self); end; end`
- `vendor/rails/activesupport/lib/active_support/core_ext/array/inquiry.rb:15-17`
  — `class Array; def inquiry; ActiveSupport::ArrayInquirer.new(self); end; end`

trails puts both at the bottom of the class files instead
(`packages/activesupport/src/string-inquirer.ts`,
`packages/activesupport/src/array-inquirer.ts`). `pnpm parity:api:extra
--package activesupport` reports each as `moved` — matched to a Ruby method, but
in the wrong file — which is the measurement of exactly this.

The misplacement has a second, visible cost. Both Ruby methods are named
`inquiry`, and because they share one TS file namespace per package barrel, the
index cannot re-export both under the Rails name: PR #6649 had to write
`export { ArrayInquirer, inquiry as arrayInquiry } from "./array-inquirer.js"`
(`packages/activesupport/src/index.ts:549`), so the public surface carries a name
Rails does not have. Splitting the files removes the collision at its source —
`core-ext/string/inquiry.ts` and `core-ext/array/inquiry.ts` are different
modules, each free to export `inquiry`.

`array-utils-splits-into-rails-core-ext-array-files` (0023, draft) is the
sibling story for the rest of the `core_ext/array` split; this one is scoped to
`inquiry` only and should be sequenced with it rather than duplicating its work.

## Converged shape

- `packages/activesupport/src/core-ext/string/inquiry.ts` exporting `inquiry`
  (the `this`-typed mixin idiom, unchanged), importing `StringInquirer`.
- `packages/activesupport/src/core-ext/array/inquiry.ts` exporting `inquiry`,
  importing `ArrayInquirer`.
- `string-inquirer.ts` / `array-inquirer.ts` keep only the classes, matching
  `string_inquirer.rb` / `array_inquirer.rb`.
- The index re-exports both under the Rails name; the `arrayInquiry` alias is
  retired and its call sites updated. Check `packages/activesupport/src/index.ts`
  for how sibling core_ext files with colliding basenames are already exported
  before choosing the spelling.
- Watch for a new package subpath: per
  `project_new_package_subpath_needs_four_registrations`, a new cross-package
  subpath needs the vitest alias plus BOTH dx-test tsconfigs, and `pnpm
typecheck` hides a miss.

## Acceptance criteria

- `pnpm parity:api:extra --package activesupport` no longer reports `inquiry` as
  `moved` for either file.
- No `arrayInquiry` name remains in the public surface; callers updated.
- `array_inquirer_test.rb` and `string_inquirer_test.rb` stay at 0
  assertion-count / 0 kind / 0 value.
