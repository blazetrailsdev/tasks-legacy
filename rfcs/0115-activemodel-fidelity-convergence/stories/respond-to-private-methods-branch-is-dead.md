---
title: "respond_to?'s include_private_methods branch cannot fire"
status: in-progress
updated: 2026-08-24
rfc: "0115-activemodel-fidelity-convergence"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 140
priority: null
pr: 7012
claim: "2026-08-24T23:18:09Z"
assignee: "move-ts-only-extras-out-of-mirrored-activemodel-serialization-test-file"
blocked-by: null
closed-reason: null
---

## Context

`ActiveModel::AttributeMethods#respond_to?`
(`vendor/rails/activemodel/lib/active_model/attribute_methods.rb:528-539`) has
three branches:

```ruby
def respond_to?(method, include_private_methods = false)
  if super
    true
  elsif !include_private_methods && super(method, true)
    false
  else
    !matched_attribute_method(method.to_s).nil?
  end
end
```

PR #7000 ported it to `packages/activemodel/src/attribute-methods.ts` with all
three branches, spelling Ruby's two `super` sends as sends to the aliased
original, `isRespondToWithoutAttributes` (attribute_methods.rb:527,
`alias :respond_to_without_attributes? :respond_to?`).

**The middle branch is dead.** `isRespondToWithoutAttributes`
(`packages/activemodel/src/attribute-methods.ts:221-229`) accepts
`includePrivateMethods` and then discards it:

```ts
export function isRespondToWithoutAttributes(
  this: object,
  method: string,
  includePrivateMethods: boolean = false,
): boolean {
  void includePrivateMethods;
  return method in (this as Record<string, unknown>);
}
```

Both sends therefore answer identically, so `super(method, true)` can never be
true when `super` was false, and `respond_to?` can never return `false` from
that arm — it always falls through to `matched_attribute_method`. Rails uses
that arm to answer `false` for a name that exists only as a PRIVATE method.

This predates #7000 (the discard was already there), but #7000 is what made it
load-bearing: before, `Model#respondTo` was a hand-rolled probe that never
consulted the alias at all.

## Converged shape

trails has no Ruby-private methods, which is why the parameter was discarded.
Either:

- establish what "private" means for a TS receiver (the `rails-private-methods`
  eslint manifest already tracks Rails-private names per file — see
  `eslint/rails-private-methods.json`, 609 files / 10687 names) and have
  `isRespondToWithoutAttributes` honour the flag against it, so the middle
  branch can fire; or
- if no TS notion of private can be made to carry it, converge by DELETING the
  dead arm rather than leaving a branch that reads as ported but cannot run,
  and justify the two-branch shape at the call site with the Rails cite.

The first is the convergence; the second is the fallback and needs the
`@missingRailsCall` / call-site justification discipline.

## Acceptance criteria

- `respond_to?`'s middle branch either fires for a genuinely private name, or is
  removed with a call-site justification citing attribute_methods.rb:528-539.
- `isRespondToWithoutAttributes` no longer accepts a parameter it discards.
- `pnpm parity:api:calls` / `:args` clean; no new baseline row.
