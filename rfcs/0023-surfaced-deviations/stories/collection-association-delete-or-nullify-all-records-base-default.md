---
title: "collection-association-delete-or-nullify-all-records-base-default"
status: draft
updated: 2026-08-11
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`CollectionAssociation#deleteOrNullifyAllRecords`
(`packages/activerecord/src/associations/collection-association.ts:551`) has a
base implementation: delete iff `method === "deleteAll"`, else nullify.

Rails has no such base method. `def delete_or_nullify_all_records` appears only
in `vendor/rails/activerecord/lib/active_record/associations/has_many_association.rb:120-122`
and `vendor/rails/activerecord/lib/active_record/associations/has_many_through_association.rb:136-138`;
`collection_association.rb` only _calls_ it (`:163`), i.e. it is an abstract
method on the base in Rails' shape.

Surfaced while reviewing PR #6387 (story
`call-args-ar-collection-proxy-delete-all-delegation`), which routed
`CollectionProxy#deleteAll` through this dispatch but did not touch the base
body — only its return type (`void` → `number`).

## Acceptance criteria

1. Establish whether every concrete `CollectionAssociation` subclass that can
   reach `deleteAll` overrides `deleteOrNullifyAllRecords` (`HasManyAssociation`
   and `HasManyThroughAssociation` do; check HABTM and any others).
2. If so, delete the trails-only base body so the base matches Rails' abstract
   shape (a declaration/throw, per the settled trails idiom for Rails abstract
   methods).
3. If some subclass genuinely relies on it, converge that subclass instead by
   giving it the Rails override, then remove the base body.
4. `pnpm parity:api:calls`, `pnpm parity:api:calls:args` and
   `pnpm parity:api:extra --package activerecord` stay green; association suites
   pass.
