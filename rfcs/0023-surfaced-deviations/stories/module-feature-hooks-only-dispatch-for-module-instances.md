---
title: "include/prepend/extend must honour append_features hooks on every module shape"
status: draft
updated: 2026-08-13
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activesupport"
deps: []
deps-rfc: []
est-loc: 90
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

PR #6461 taught `include()` / `prepend()` / `extend()` in
`packages/activesupport/src/include.ts` to dispatch through a module's own
`appendFeatures` / `prependFeatures` / `extended` hook, which is how Ruby defines
those three methods — `Deprecation::DeprecatedConstantProxy` overrides all three
to warn before mixing in
(`vendor/rails/activesupport/lib/active_support/deprecation/proxy_wrappers.rb:156-169`).

The dispatch is deliberately narrowed: `featureHook()` returns a hook only when
`mod instanceof Module`. A plain-object module (the far more common trails module
shape) that defines an `appendFeatures` method is silently treated as if it had
none, and the method is copied onto the class as an ordinary member instead of
being called. Ruby draws no such distinction — `append_features` is honoured on
every module.

The narrowing was chosen because a plain-object module with a coincidentally
same-named key would otherwise change behaviour, but that is a guess about
existing modules, not a language constraint.

## Converged shape

Honour the three hooks on every module shape `include()` / `prepend()` /
`extend()` accept, as Ruby does. If a real collision exists in the tree, fix that
module rather than keeping the type guard — grep the plain-object modules for the
three names first and report the count in the PR.

## Acceptance criteria

- `featureHook()`'s `instanceof Module` guard is gone; the hooks dispatch for
  plain-object and class modules too.
- Any colliding member found in the tree is renamed, not worked around.
- `include.test.ts` and `proxy-wrappers.test.ts` stay green.
