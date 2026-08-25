---
title: "only: filter accepts only classes; Rails accepts modules via model <= klass"
status: draft
updated: 2026-08-17
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "globalid"
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

Ruby's `only:` filter accepts a **Module**, not just a Class:
`GlobalID::Locator.find_allowed?` tests `model <= klass`
(`vendor/globalid/lib/global_id/locator.rb:207-213`), and `Module#<=` is true
when the class includes the module. Rails exercises this with four tests in
`global_id_test.rb`:

- `find with module` (`:80-89`) — `only: GlobalID::Identification`,
  `only: ActiveModel::Model`
- `find with module no match` (`:91-96`) — `only: Enumerable`,
  `only: Forwardable`, `only: Base64`
- `find with multiple module` (`:114-126`)
- `find with multiple module no match` (`:128-133`)

All four are registered unported in
`scripts/parity/unported-files/unscoped.ts:691-703`, with the reason that
"TypeScript has no module-include relationship for classes; the class-based
equivalents cover the same routing logic."

That reason deserves re-derivation rather than inheritance. trails **does** have
a settled mixin mechanism — `include()` / `Included<>` from
`@blazetrails/activesupport` (see `activesupport/src/include.ts`, and CLAUDE.md's
"Module mixins" section) — and `GlobalID::Identification` itself is ported as
`packages/globalid/src/identification.ts` with `this`-typed functions. If a
mixin leaves any runtime trace (a marker on the prototype, a registry entry),
`findAllowed` can test membership and the four tests become portable.

Trails' `Locator.findAllowed` (`packages/globalid/src/locator.ts:355-364`)
currently handles only the class case: identity or `fn.prototype instanceof cFn`.

## Converged shape

Extend `findAllowed` so `only:` accepts a mixin as well as a constructor,
mirroring `model <= klass` for both arms. The exact mechanism depends on what
`include()` records — investigate that first; if mixins genuinely leave no
inspectable trace, the honest outcome is to add one (Ruby has
`Module#included_modules`, so a registry is not a trails invention) or to block
the story with that specific finding.

Note the class-based arm must keep working unchanged: `find with class` and
`find with multiple class` were re-ported in PR #6651 and pass.

## Acceptance criteria

- `only:` accepts a mixin and filters by inclusion, mirroring `locator.rb:207-213`.
- The four `global_id_test.rb` module tests are ported and passing.
- Their row is removed from `scripts/parity/unported-files/unscoped.ts`.
- `pnpm parity:test --package globalid` does not drop from 131/131.
