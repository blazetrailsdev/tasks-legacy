---
title: "Inline coerceToRecords into delete_or_destroy as Rails does"
status: draft
updated: 2026-08-13
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
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

`CollectionAssociation#deleteOrDestroy`
(`packages/activerecord/src/associations/collection-association.ts:1079-1120`)
delegates its id coercion to a trails-only private helper, `coerceToRecords`
(`:1131-1152`). Rails has no such method: `delete_or_destroy` does the coercion
inline —

```ruby
def delete_or_destroy(records, method)
  return if records.empty?
  records = find(records) if records.any? { |record| record.kind_of?(Integer) || record.kind_of?(String) }
  records = records.flatten
  ...
```

(`vendor/rails/activerecord/lib/active_record/associations/collection_association.rb:385-397`).

The helper also carries two shapes Rails does not have: a `reflection.options.through`
branch that resolves ids against the loaded target instead of `find`, and a bare
`throw new Error("Couldn't find …")` (already filed as
`coerce-to-records-through-branch-raises-bare-error`).

PR #6442 made the helper `Promise<Base[]> | Base[]` alongside the rest of the
chain but did not touch its existence.

## Converged shape

Inline the coercion into `deleteOrDestroy` as the single `records = find(records)
if records.any? { … }` line Rails has, and delete `coerceToRecords`. The through
branch needs a decision first: either `find` works across the join for a through
association (in which case the branch is dead and goes away with the helper) or
it does not, and the divergence moves into the ported `find` where Rails' HMT
`find` lives (`has_many_through_association.rb`), not into a new private method.

## Acceptance criteria

- [ ] `coerceToRecords` is gone; `deleteOrDestroy` coerces inline as Rails does.
- [ ] `pnpm parity:api:extra --package activerecord` does not grow.
- [ ] `packages/activerecord/src/associations` green on all adapter lanes.
