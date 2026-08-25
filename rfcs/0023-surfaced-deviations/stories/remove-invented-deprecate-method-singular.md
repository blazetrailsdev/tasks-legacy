---
title: "Deprecation#deprecateMethod (singular) is invented; Rails has only deprecate_methods"
status: draft
updated: 2026-08-13
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activesupport"
deps: []
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

Surfaced by PR #6456, which ported `deprecate_methods`
(`activesupport/lib/active_support/deprecation/method_wrappers.rb:35-67`) into
`packages/activesupport/src/deprecation/method-wrappers.ts` and mixed it onto
`Deprecation`.

`Deprecation#deprecateMethod` (singular) at
`packages/activesupport/src/deprecation.ts:457-465` is a trails invention with no
Rails counterpart — Rails has only the plural `deprecate_methods`, whose message
comes from `deprecated_method_warning` (reporting.rb:112-119) rather than from a
mandatory caller-supplied string. The invented method also calls `warn(message)`
directly, so it skips `deprecation_warning` and therefore the
`"<name> is deprecated and will be removed from <gem> <horizon>"` message Rails
generates.

It has ~10 call sites, all in tests (`deprecation.test.ts`,
`hwia-module-string.test.ts`).

## Converged shape

```ruby
# method_wrappers.rb:35-67
deprecator.deprecate_methods(Fred, :aaa, bbb: :zzz, ccc: "use Bar#ccc instead")
```

Delete `deprecateMethod` and route every call site through the ported
`deprecateMethods(target, name)` / `deprecateMethods(target, { name: message })`,
whose warning text is Rails'.

## Acceptance criteria

- `Deprecation#deprecateMethod` is gone; `pnpm parity:api:extra --package activesupport`
  no longer reports it.
- Test call sites use `deprecateMethods`; any test asserting the old
  caller-supplied message asserts Rails' `deprecated_method_warning` text instead
  (test names unchanged).
