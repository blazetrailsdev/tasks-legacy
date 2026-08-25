---
title: "CollectionProxy is built for declared singular names, which Rails never does"
status: draft
updated: 2026-08-18
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 260
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

trails' `association(record, name)` factory
(`packages/activerecord/src/associations.ts:1789-1867`) builds a
`CollectionProxy` for ANY declared association name, including a declared
SINGULAR one — `association(human, "autosaveFace")` in
`packages/activerecord/src/associations/inverse-associations.test.ts:1160-1174`
is a has_one reached this way, and `collection-proxy.ts:976-990` has explicit
`hasOne` / `hasOneThrough` arms in the through-write path to serve it.

Rails has no such object. `CollectionProxy.create(klass, association)` is
reached only from `CollectionAssociation#reader`
(`vendor/rails/activerecord/lib/active_record/associations/collection_association.rb:33-42`);
a has_one/belongs_to reader returns the record itself via
`SingularAssociation#reader`
(`vendor/rails/activerecord/lib/active_record/associations/singular_association.rb:11-17`).

The cost was paid in PR #6685 (`retire-collection-proxy-own-seat`): the proxy
now holds its association the way `collection_proxy.rb:31-35` does, but the
singular case cannot share the `SingularAssociation`'s `@target` (one record,
not an array — folding a collection array into it boxes the record, which is
what reddened `inverse-associations.test.ts` "has_one createBang stores a
single record as target, not an array" and `associations.test.ts` "preload with
available records with through association" in the shape rejected by #6683).
So `CollectionProxy`'s constructor still carries a branch building a standalone
`new CollectionAssociation(record, assocDef)` purely for proxies over singular
names (`collection-proxy.ts`, `_targetAssociation`).

## Converged shape

- `association(record, name)` returns a `CollectionProxy` only for a collection
  macro (`hasMany` / `hasAndBelongsToMany`); a singular name resolves to its
  `SingularAssociation` (`Base#association`, `associations.rb:290-296`) and its
  callers read the record through `singular_association.rb:11-17`.
- The `hasOne` / `hasOneThrough` arms in `collection-proxy.ts:976-990` and the
  standalone-`CollectionAssociation` branch in the constructor are deleted with
  it — the proxy then always holds the owner's real `CollectionAssociation`,
  exactly as `collection_proxy.rb:31-35` hands it in.
- Test call sites that pass a singular name to the factory move to the singular
  reader.

## Acceptance criteria

- [ ] `association(record, name)` refuses / does not build a `CollectionProxy`
      for a declared singular macro.
- [ ] `CollectionProxy`'s constructor holds the owner's `CollectionAssociation`
      unconditionally; no standalone-association fallback branch remains.
- [ ] The two tests named above stay green, as do the association, preload,
      autosave and nested-attributes suites on SQLite, PostgreSQL and
      MySQL/MariaDB.
