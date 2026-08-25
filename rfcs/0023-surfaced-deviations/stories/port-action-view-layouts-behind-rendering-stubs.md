---
title: "port-action-view-layouts-behind-rendering-stubs"
status: draft
updated: 2026-07-30
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "actionview"
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

`packages/actionview/src/rendering.ts:28-51` declares three interfaces marked
`@internal stub - real impl in Phase 4`: `Rendering`, `Layouts` and
`LayoutsClass`. `Layouts` is a stand-in for `ActionView::Layouts`
(`vendor/rails/actionview/lib/action_view/layouts.rb:205`), which trails has not
ported — there is no `packages/actionview/src/layouts.ts`.

Found by the RFC 0080 audit of `moved` interface declaration names
(`audit-moved-interface-declaration-names`), which tagged `Layouts`
`@noRailsEquivalent CONVERGEABLE (story: <this story>)`. The sibling stubs are
novel names and so are exempt by kind; they should go the same way.

## Acceptance criteria

- `ActionView::Layouts` is ported into `packages/actionview/src/layouts.ts`
  following the Rails layout (`_layout_for`, `_layout_for_rendering`, and the
  `layout` class method).
- `rendering.ts` consumes the ported module and the stub interfaces plus the
  `@noRailsEquivalent` tag are deleted.
- `pnpm parity:api:extra` exits 0 (no stale tag).
