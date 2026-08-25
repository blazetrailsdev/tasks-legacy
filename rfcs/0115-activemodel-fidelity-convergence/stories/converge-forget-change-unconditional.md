---
title: "converge-forget-change-unconditional"
status: claimed
updated: 2026-08-25
rfc: "0115-activemodel-fidelity-convergence"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: null
claim: "2026-08-25T01:39:08Z"
assignee: "converge-forget-change-unconditional"
blocked-by: null
closed-reason: null
---

## Context

`AttributeMutationTracker#forget_change`
(`vendor/rails/activemodel/lib/active_model/attribute_mutation_tracker.rb:54-57`)
is two unconditional statements, in this order:

```ruby
def forget_change(attr_name)
  attributes[attr_name] = attributes[attr_name].forgetting_assignment
  forced_changes.delete(attr_name)
end
```

`packages/activemodel/src/attribute-mutation-tracker.ts:104-110` diverges twice:
it deletes from `forcedChanges` **first**, and it wraps the attribute rewrite in
an `isKey(attrName)` guard Rails does not have. Rails' `[]` returns the `Null`
default for an unknown name (`attribute_set.rb:16-18`), so Rails writes back a
`forgetting_assignment` of the Null attribute where trails writes nothing.

Surfaced by the reviewer on PR #7021 (RFC 0115
`retire-attribute-set-map-adapter-surface`), which only renamed the guard's
`.has` call to `.isKey` and left the guard itself alone as out of scope.

## Acceptance criteria

- `forgetChange` is the two unconditional statements, in Rails' order: the
  attribute rewrite through `getAttribute`/`set`, then
  `forcedChanges.delete(attrName)`.
- No `isKey` guard remains; an unknown name goes through the `Null` default the
  way Rails' `[]` does.
- `pnpm parity:api:calls` and `pnpm parity:api:calls:args` stay clean (a row for
  this method, if one exists, shrinks rather than grows).

## Verification

```bash
pnpm vitest run packages/activemodel/src/dirty.test.ts packages/activemodel/src/dirty-mutations.test.ts packages/activerecord/src/dirty.test.ts
```
