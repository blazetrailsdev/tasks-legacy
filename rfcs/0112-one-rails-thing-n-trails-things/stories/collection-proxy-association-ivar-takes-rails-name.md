---
title: "CollectionProxy's @association ivar cannot take the Rails name while the AssociationRelation this-alias holds it"
status: in-progress
updated: 2026-08-25
rfc: "0112-one-rails-thing-n-trails-things"
cluster: duplicate-bodies
packages: []
deps: []
deps-rfc: []
est-loc: 180
pr: 7030
claim: "2026-08-25T12:59:40Z"
assignee: "collection-proxy-association-ivar-takes-rails-name"
blocked-by: null
closed-reason: null
---

## Context

Rails' `CollectionProxy` holds `@association`
(`vendor/rails/activerecord/lib/active_record/associations/collection_proxy.rb:31-35`),
and `target` / `loaded?` (`:33`, `:53`) read it. PR #6685 gave trails' proxy
that ivar — resolved once in the constructor — but had to spell it
`_targetAssociation`, because `_association` on `CollectionProxy`
(`packages/activerecord/src/associations/collection-proxy.ts`) is already taken
by a trails-only `get _association(): this` self-alias.

That alias exists so `AssociationRelation` bodies can run with the proxy as
receiver (`AssociationRelation.prototype.toArray.call(proxy)`): they read
`this._association.owner` / `.reflection` and, at
`packages/activerecord/src/association-relation.ts:138,171`, `this._association`
as the proxy itself (`.build(...)`, the `_collectionProxies` lookup). So the
Rails ivar cannot take the Rails name while the alias occupies it.

Rails has no self-alias: `AssociationRelation#initialize` is handed the
association (`relation/delegation.rb:138-140`,
`vendor/rails/activerecord/lib/active_record/association_relation.rb:5-9`), and
`proxy_association` returns it.

## Converged shape

- `CollectionProxy`'s Rails ivar takes the Rails name (`_association`), and the
  `this`-alias getter is deleted.
- `AssociationRelation` bodies reach the proxy by the seat they are actually
  handed rather than by a name the receiver has to shadow — either always
  constructing the relation with its association (Rails' shape) or reading
  `proxyAssociation` (`collection_proxy.rb`'s reader,
  `collection-proxy.ts:2573`), which both classes already expose.

## Acceptance criteria

- [ ] `CollectionProxy` holds Rails' `@association` under the name
      `_association`; `_targetAssociation` no longer exists.
- [ ] No `get _association(): this` self-alias remains on `CollectionProxy`.
- [ ] `association-relation`, associations, autosave and nested-attributes
      suites stay green on SQLite, PostgreSQL and MySQL/MariaDB.
- [ ] No new `parity:api:extra` surface; `parity:api:calls` / `:args` green.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
