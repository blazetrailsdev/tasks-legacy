---
title: "CollectionProxy#deleteAll re-looks the association up through the owner instead of reading the @association seat"
status: ready
updated: 2026-08-25
rfc: "0112-one-rails-thing-n-trails-things"
cluster: null
packages: []
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

Rails' `CollectionProxy#delete_all` is
`@association.delete_all(dependent).tap { reset_scope }`
(`vendor/rails/activerecord/lib/active_record/associations/collection_proxy.rb:474-476`)
— it reads the SAME `@association` ivar (`:31-35`) that `target` / `loaded?`
read (`:33`, `:53`).

trails' `CollectionProxy#deleteAll`
(`packages/activerecord/src/associations/collection-proxy.ts`, the `deleteAll`
body) instead re-looks the association up through the owner:

```ts
await (
  this._record.association(this._assocName) as unknown as {
    deleteAll(dependent?: string): Promise<number>;
  }
).deleteAll(dependent);
```

So one Rails `@association` read has two trails spellings in the same class.
Its siblings already use the seat: `delete` / `destroy` / `isInclude` go through
`this._collectionAssociation()`, and PR #7030 gave the ivar the Rails name
`_association`, so the seat is now spelled exactly as Rails spells it.

The owner lookup is also not equivalent for the proxy trails builds for a
declared SINGULAR name: the constructor gives such a proxy a collection seat of
its own (`new CollectionAssociation(record, assocDef)`), which
`record.association(name)` does not return.

## Converged shape

`deleteAll` reads the `@association` seat like its siblings — the
`_collectionAssociation()` / `_association` read — and drops the inline
structural cast, leaving `collection_proxy.rb:474-476`'s one-liner shape.

## Acceptance criteria

- [ ] `CollectionProxy#deleteAll` reaches the association through the
      `@association` seat (`_association` / `_collectionAssociation()`), not
      `this._record.association(this._assocName)`.
- [ ] The `as unknown as { deleteAll(...) }` structural cast is gone.
- [ ] `pnpm parity:api:calls` / `:args` / `pnpm parity:api:extra --package activerecord` green;
      the association, autosave and nested-attributes suites pass on SQLite, PostgreSQL and MySQL/MariaDB.
