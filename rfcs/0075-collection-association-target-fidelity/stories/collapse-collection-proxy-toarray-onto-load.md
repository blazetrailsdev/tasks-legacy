---
title: "collapse-collection-proxy-toarray-onto-load"
status: blocked
updated: 2026-08-22
rfc: "0075-collection-association-target-fidelity"
cluster: null
packages: []
deps:
  - retire-collection-proxy-query-executor-flag
  - hoist-mid-load-guard-to-doasyncfindtarget-callers
  - resolve-nested-through-unpersisted-owner-test-against-rails
deps-rfc: []
est-loc: null
priority: null
pr: null
claim: "2026-08-20T02:22:31Z"
assignee: "collapse-collection-proxy-toarray-onto-load"
blocked-by: "Verified on branch: Rails' literal load_target shape (toArray => load(); load() guards the query with find_target?/!isNullScope(), collection_association.rb:272-279) FIXES the autosave 'parent should save children record with foreign key validation set in before save callback' arm, and reds exactly one test: has-many-through-associations.test.ts 'nested has many through association with unpersisted parent instance' (:2337). That test has no Rails counterpart (no 'unpersisted parent' in vendor/rails/activerecord/test) and asserts behaviour Rails does not have: for post.subscriptions (through :books, itself through :author) ThroughAssociation#foreign_key_present? is false (through_association.rb:90-94, through_reflection.belongs_to? false) and owner.new_record? is true, so find_target? (association.rb:320-322) is false and Rails returns []. The only thing keeping that test green is the cache-bypassing toArray arm this story deletes. Unblocked by RFC 0075's retire-collection-proxy-query-executor-flag (draft) + hoist-mid-load-guard-to-doasyncfindtarget-callers (ready), so the null-scope path stops querying, plus a decision on that trails-only test. Sibling stories retire-collection-proxy-bang-finder-and-first-or-overrides and converge-join-constraints-scope-join-sources-inline shipped in the same PR without it."
closed-reason: null
---

## Context

`CollectionProxy#toArray` still carries one arm `load()` does not, so Rails'
single `to_a` is `records` is `load_target` body
(`collection_proxy.rb:1024-1026`, `:44-46`) is two bodies in trails:

```ts
async toArray(): Promise<T[]> {
  if (!this._targetLoaded && this.isNullScope()) {
    const results = await this._execLoad();
    return this._collectionAssociation().mergeTargetLists(results, this._target) as T[];
  }
  return this.load();
}
```

(`packages/activerecord/src/associations/collection-proxy.ts`, added by #6755.)

PR #6755 tried to collapse the two and had to back it out — both directions
regress, and the pair of failures pins the constraint precisely:

- **Moving the arm into `load()`** reds
  `autosave-association.test.ts` >
  `TestDefaultAutosaveAssociationOnAHasManyAssociation` >
  `parent should save children record with foreign key validation set in before
save callback` ("expected [] to not have a length of +0"). The arm returns
  merged records without writing `_target` / `_targetLoaded`, and in `load()`
  it also catches `loadTarget()` — which `CollectionAssociation#concat` calls on
  a new-record owner before appending (`collection_association.rb:439-446`) and
  needs to cache. The `beforeSave` push on `NewlyContractedCompany`
  (`test-helpers/models/company.ts:605`) is then dropped.
- **Deleting the arm outright** reds
  `has-many-through-associations.test.ts` >
  `HasManyThroughAssociationsTest` >
  `nested has many through association with unpersisted parent instance`
  ("expected [ 1 ] to include 2", `:2363`). A null-scope through collection has
  to re-traverse the in-memory chain on each read rather than serve a cached
  subset.

Rails has neither problem because `load_target`
(`collection_association.rb:272-279`) skips the query entirely when
`!find_target?` and still reaches `loaded!` — it never runs a query it then
refuses to cache. trails' `_execLoad` / `_findTargetViaAssociation` query on the
null scope instead, which is what forces the no-cache arm.

So the collapse is gated on the `find_target?` / `_queryExecutor` residue that
RFC 0075 owns — specifically
`retire-collection-proxy-query-executor-flag`
and `.../hoist-mid-load-guard-to-doasyncfindtarget-callers`.

## Acceptance criteria

- `CollectionProxy#toArray` and `#load` share one body, matching Rails' single
  `to_a` / `records` / `load_target` chain.
- The null-scope path skips the query rather than running one it cannot cache,
  mirroring `load_target`'s `if find_target?` guard
  (`collection_association.rb:272-279`), and still reaches the `loaded!`
  equivalent.
- Both tests named above keep passing, unrenamed:
  `parent should save children record with foreign key validation set in before save callback`
  and `nested has many through association with unpersisted parent instance`.
- `pnpm parity:api:calls` / `:args` add zero rows.
