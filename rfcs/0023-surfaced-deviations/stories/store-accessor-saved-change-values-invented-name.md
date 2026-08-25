---
title: "store accessor savedChangeTo<Key>Values is an invented name with no Rails counterpart"
status: ready
updated: 2026-07-27
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
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

Surfaced while fixing the null/undefined deviation in #4982.

Rails defines two store-accessor saved-change methods
(`vendor/rails/activerecord/lib/active_record/store.rb:163,169`):

- `saved_change_to_#{key}?` — predicate, returns `true`/`false`
- `saved_change_to_#{key}` — value form, returns `[prev, next]` or `nil`

Ruby distinguishes them by the `?` suffix. TypeScript has no such suffix, so
`packages/activerecord/src/store.ts:431,436` renames the pair:

- `savedChangeTo<Key>()` — takes the predicate (Rails' `?` form)
- `savedChangeTo<Key>Values()` — invented name for Rails' value form

The `Values` suffix exists nowhere in Rails. The existing comment at
`store.ts:428-430` justifies it as parallel to Model's
`savedChangeToAttributeValues`, so the invention is at least consistent
within trails — but it is still a public-surface name with no Rails
counterpart, and `parity:api` cannot map it.

Note the same `?`-collision problem is already solved elsewhere in the
codebase for generated dirty methods; the fix should follow whatever
convention that precedent sets rather than inventing a third one. Worth
confirming which surface is canonical before changing a public method name.

## Acceptance criteria

- [ ] Decide and document the canonical trails naming for Ruby `foo?` /
      `foo` method pairs on generated store accessors
- [ ] `savedChangeTo<Key>Values` either converges to that convention or is
      recorded as a deliberate, documented deviation with the reason
- [ ] `parity:api` mapping updated (or a SKIP_GROUPS entry added with the
      reason) so the pair is not silently unmapped
- [ ] `store.test.ts` coverage follows the resulting names
