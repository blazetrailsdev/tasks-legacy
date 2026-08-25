---
title: "has_one setOwnerAttributes must resolve the owner PK via reflection.active_record_primary_key"
status: ready
updated: 2026-07-27
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
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

Surfaced while deleting the `set*` engine writers in #5355.

`HasOneAssociation#setOwnerAttributes`
(`packages/activerecord/src/associations/has-one-association.ts:496-499`)
resolves the owner-side primary key as:

```ts
const ctor = (this.owner as any).constructor;
const configuredPk = this.reflection.options.primaryKey ?? ctor.primaryKey ?? "id";
```

That reads the primary key off the **owner instance's class** and the
lightweight `AssociationDefinition.options`. Rails uses
`reflection.active_record_primary_key`
(`vendor/rails/activerecord/lib/active_record/associations/has_one_association.rb`
-> `association.rb#set_owner_attributes`), which resolves through the
registered reflection (option -> `query_constraints_list` -> declaring class PK).

PR #5355 already converged the **foreign-key** half of the same method:
`foreignKeyColumns` now consults the rich reflection via
`ctor._reflectOnAssociation(name)?.foreignKey`, mirroring
`reflection.foreign_key`. The primary-key half was left on the owner-class
path, so the two halves of one method now source from different places.

The collection sibling already does this correctly and documents why —
`CollectionAssociation` (`collection-association.ts:740-753`) resolves the rich
reflection specifically because `this.reflection` "has no
`activeRecordPrimaryKey` getter — so `foreignKeyPresentFor` would fall back to
`"id"` and report a custom-PK owner's FK absent". has_one has the identical
exposure.

Low observed blast radius today (an STI subclass inherits `primaryKey`, so the
common case coincides), but a custom-PK or `query_constraints` owner on a
has_one writes the wrong PK value into the target's FK.

## Acceptance criteria

- `HasOneAssociation#setOwnerAttributes` resolves the owner primary key
  through the rich reflection's `activeRecordPrimaryKey`, the same way
  `CollectionAssociation` does, rather than `owner.constructor.primaryKey`.
- Keep the existing fallback for an unregistered/undeclared association.
- A test with a custom-PK has_one owner (mirror the Rails model the collection
  path uses, e.g. `Subscriber`/`subscriptions`, in its has_one analogue) pins
  the PK value written into the target FK. Test name matches Rails verbatim if
  a Rails test covers it; otherwise it is a `.trails.test.ts` extra.
- Existing has_one suites stay green.
