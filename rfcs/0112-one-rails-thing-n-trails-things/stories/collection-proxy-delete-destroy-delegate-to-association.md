---
title: "CollectionProxy#delete/#destroy should delegate to the association like Rails"
status: done
updated: 2026-08-25
rfc: "0112-one-rails-thing-n-trails-things"
cluster: duplicate-bodies
packages: []
deps: []
deps-rfc: []
est-loc: 220
pr: 6435
claim: "2026-08-25T12:59:40Z"
assignee: "collection-proxy-association-ivar-takes-rails-name"
blocked-by: null
closed-reason: null
---

## Context

Now that `CollectionProxy#deleteAll` delegates (PR #6387), its siblings are the
last reimplementations of the same dispatch inside the proxy.

Rails:

- `collection_proxy.rb:620-622` — `def delete(*records) @association.delete(*records).tap { reset_scope } end`
- `collection_proxy.rb:692-694` — `def destroy(*records) @association.destroy(*records).tap { reset_scope } end`

`CollectionAssociation#delete` / `#destroy`
(`collection_association.rb:186-196` → `delete_or_destroy` → `remove_records` →
`delete_records`) already exist in trails at
`packages/activerecord/src/associations/collection-association.ts:568-590`, and
`HasManyAssociation#deleteRecords` /
`HasManyThroughAssociation#deleteRecords` carry the real strategies.

trails instead reimplements the whole path in
`packages/activerecord/src/associations/collection-proxy.ts` (`_deleteStrategy()`
around :2551, the `removeRecords` body around :2440-2540, `_decrementCounterCache`
around :2597), including a proxy-local `:destroy`/`:delete`/`:delete_all`/
`"deleteAll"` mapping that now disagrees with the association layer's public
`deleteAll` vocabulary.

## Acceptance criteria

1. `CollectionProxy#delete` and `#destroy` are the Rails one-liners: delegate to
   `association.delete(...)` / `association.destroy(...)`, then `resetScope()`
   (plus the proxy-local target replay `deleteAll` already documents, since the
   trails proxy keeps its own target copy).
2. The strategy mapping, the counter-cache decrement and the scoped bulk
   delete/nullify live in `CollectionAssociation` / `HasManyAssociation` /
   `HasManyThroughAssociation` only — `_deleteStrategy` and
   `_decrementCounterCache` are deleted from the proxy.
3. Any resulting call/call-arg baseline rows are converged, not added.
4. `pnpm parity:api:calls`, `pnpm parity:api:calls:args`,
   `pnpm parity:api:extra --package activerecord` green; the association suites
   pass.

## Absorbed: `collection-proxy-clear-delegates-to-delete-all`

Merged in during the RFC 0023 triage pass (2026-08-18). Original title: "CollectionProxy#clear should be delete_all + self"

### Context

Rails' `CollectionProxy#clear` is two lines (`collection_proxy.rb:1066-1069`):

```ruby
def clear
  delete_all
  self
end
```

trails' `packages/activerecord/src/associations/collection-proxy.ts` `clear()`
(around :2680) is ~80 lines: it re-derives the dependent strategy
(`dep === "destroy" || "delete" || "delete_all" || "deleteAll"`), branches on
`_isThrough` and `_relationStateDiverged()`, reaches into the association's
`loadTarget`/`deleteRecords` directly, calls `_buildNullifyUpdates()` /
`super.updateAll` / `scope().deleteAll`, and decrements the counter cache
itself — i.e. the same dispatch `deleteAll` reimplemented a second time.

PR #6387 converged `deleteAll` to `@association.delete_all(dependent).tap {
reset_scope }` (`collection_proxy.rb:474-476`), so `clear` can now simply call
it. `clear` also returns `self` in Rails; trails returns `void`.

### Acceptance criteria

1. `CollectionProxy#clear` is `await this.deleteAll(); return this;` — no
   strategy derivation, no through branch, no counter-cache handling of its own.
2. The `isNullScope()` short-circuit is either shown to be covered by the
   association path (`scope.none!`, `collection_association.rb:300-305`) or kept
   with a Rails cite at the call site.
3. `clear` returns the proxy (Rails' `self`), and the existing `clear` tests
   still pass.
4. `pnpm parity:api:calls` / `pnpm parity:api:calls:args` green; no new baseline
   rows.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
