---
title: "Retire the anonymous-inline reflection fallback and its two klass==null guards"
status: claimed
updated: 2026-08-25
rfc: "0112-one-rails-thing-n-trails-things"
cluster: duplicate-bodies
packages: []
deps: []
deps-rfc: []
est-loc: 140
pr: null
claim: "2026-08-25T16:50:30Z"
assignee: "converge-collection-proxy-anonymous-inline-reflection-guards"
blocked-by: null
closed-reason: null
---

## Context

Surfaced by PR #6428 (RFC 0084,
`converge-collection-proxy-rich-reflection-re-resolve`).

`CollectionProxy#reflection` now resolves the registered reflection off the
owner's class, mirroring Rails' `Association#reflection`
(`vendor/rails/activerecord/lib/active_record/associations/association.rb:16`,
handed the rich reflection at `associations.rb:290-296`). It falls back to the
thin macro definition (`_assocDef`) when the owner's class has no registered
reflection for the name — an anonymous inline association.

That fallback forces two guards Rails has no counterpart for, both in
`packages/activerecord/src/associations/collection-proxy.ts`:

- `_djarForCount`: `if (reflection.klass == null) return null;`
- `_foreignKeyPresent`: `if (reflection.klass == null) return false;`

They preserve the pre-#6428 `if (!reflection) return ...` short-circuit exactly
(macro definitions never carry `klass`; only the reflection-builder call sites
at `associations.ts:1126,1251` do), but Rails' `CollectionAssociation` bodies
have no such branch — `foreign_key_present?`
(`collection_association.rb`, `foreign_association.rb:5`) reads the reflection
unconditionally, because in Rails there is always one.

The root cause is that trails permits an association with no registered
reflection at all.

## Converged shape

An inline/anonymous association registers a reflection like every other, so
`CollectionProxy#reflection` never needs the `_assocDef` fallback and both
`klass == null` guards are deleted along with it. Same root cause as
[[converge-association-reflection-type-drop-association-definition]] and
[[converge-association-klass-to-reflection-klass-delegate]].

## Acceptance criteria

- [ ] Both `reflection.klass == null` guards in `collection-proxy.ts` deleted.
- [ ] `CollectionProxy#reflection` returns the registered reflection with no
      `?? this._assocDef` arm, or that arm is shown unreachable.
- [ ] AR association suites green on all three adapter lanes.

## Update from the 2026-08-18 RFC 0023 triage pass

**AC 1 is already met.** Neither `_djarForCount` nor `_foreignKeyPresent` exists
anywhere in `packages/` or `scripts/` any more, and `collection-proxy.ts` carries
no `reflection.klass == null` guard.

**AC 2 is what remains.** `CollectionProxy#reflection`
(`collection-proxy.ts:315-320`) still ends in the fallback arm:

```ts
const ctor = this._record.constructor as typeof Base;
return (
  (ctor._reflectOnAssociation?.(this._assocName) as AssociationDefinition | null) ?? this._assocDef
);
```

and `_assocDef` is still a field (`:190`), also read by `_callbackHost` (`:330`).
Rails' `Association#reflection` (`association.rb:16`) is handed the rich
reflection at `associations.rb:290-296` and has no fallback.

Scope this story to removing the `?? this._assocDef` arm (or demonstrating it
unreachable) and retiring the `_assocDef` field with it.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
