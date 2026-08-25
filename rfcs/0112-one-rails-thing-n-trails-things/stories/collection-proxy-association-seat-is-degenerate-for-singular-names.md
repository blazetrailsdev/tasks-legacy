---
title: "collection-proxy-association-seat-is-degenerate-for-singular-names"
status: claimed
updated: 2026-08-25
rfc: "0112-one-rails-thing-n-trails-things"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: null
claim: "2026-08-25T17:02:43Z"
assignee: "collection-proxy-association-seat-is-degenerate-for-singular-names"
blocked-by: null
closed-reason: null
---

# `CollectionProxy`'s `@association` seat is degenerate for a declared singular name

## Context

Surfaced in PR #7038 (RFC 0112,
`collection-proxy-delete-all-reads-the-association-seat`).

Rails' `CollectionProxy` is HANDED its association and holds it in
`@association` (`vendor/rails/activerecord/lib/active_record/associations/collection_proxy.rb:31-35`);
`target` (:33), `loaded?` (:53) and `delete_all` (:474-476) are all the same
one ivar read.

trails has TWO spellings of that read in the same class:

- `_association`, assigned in the constructor
  (`packages/activerecord/src/associations/collection-proxy.ts`, the
  `constructor` body):

  ```ts
  const instance = record.association(assocName) as unknown as CollectionAssociation;
  this._association = instance.isCollection()
    ? instance
    : new CollectionAssociation(record, assocDef);
  ```

- `_collectionAssociation()`, which re-looks the association up through the
  owner: `this._record.association(this._assocName)`.

PR #7038 tried to collapse the second onto the first. That reds
`inverse-associations.test.ts` >
`has_one createBang stores a single record as target, not an array` on every
adapter lane:

```text
TypeError: Cannot read properties of undefined (reading 'slice')
 ❯ drop packages/activerecord/src/ruby-drop.ts:31
 ❯ AssociationScope.getChain associations/association-scope.ts:481
 ❯ CollectionAssociation.scopeForCreate associations/association.ts:1055
```

because for a DECLARED SINGULAR name the constructor's `else` arm synthesizes
`new CollectionAssociation(record, assocDef)` — an association that is not the
one the owner holds, is not registered on the owner, and whose reflection is
not wired up, so `associationScope` walks an undefined chain.

So the seat is not usable as Rails' `@association`, and the owner lookup is not
equivalent either (it returns the SINGULAR association, which is not what a
CollectionProxy's mutations want). Both spellings are wrong in different ways;
PR #7038 kept the owner lookup because it is the one every sibling
(`delete` / `destroy` / `isInclude` / `create`) already uses.

## Converged shape

One seat, holding the association the owner holds, so `_collectionAssociation()`
is the plain `@association` read Rails makes. That means the synthesized
`new CollectionAssociation(record, assocDef)` arm goes away: either the proxy is
not built for a declared singular name at all, or the association it is handed
is the owner's, registered and reflection-wired.

## Acceptance criteria

- [ ] `CollectionProxy` holds ONE association seat; `_collectionAssociation()`
      returns it and no method re-looks it up through the owner.
- [ ] The constructor does not synthesize a `CollectionAssociation` the owner
      does not hold.
- [ ] `inverse-associations.test.ts` >
      `has_one createBang stores a single record as target, not an array`
      stays green on SQLite, PostgreSQL and MySQL/MariaDB, together with the
      association, autosave and nested-attributes suites.
