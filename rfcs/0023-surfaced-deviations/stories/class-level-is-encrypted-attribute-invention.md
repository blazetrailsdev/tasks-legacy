---
title: "Class-level isEncryptedAttribute is a trails invention; converge to encrypted_attributes membership"
status: ready
updated: 2026-07-27
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 40
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced in #5097 review (finding 1). The class-level
`isEncryptedAttribute(klass, attr)` in `packages/activerecord/src/encryption.ts`
(exported via encryption/index.ts) has no Rails counterpart — Rails' nearest
surface is `encrypted_attributes.include?(name)`
(vendor/rails/activerecord/lib/active_record/encryption/encryptable_record.rb:146-148
uses it inside the instance predicate). After #5097 its `_attributeDefinitions`
arm is dead for real Base subclasses (only the `_pendingEncryptions` arm fires;
the defs arm survives solely for mock models registered via
registerEncryptedType). Audit callers and converge to an
`_encryptedAttributes`-membership check (or retire the helper) so the invented
prototype-walk + pending/defs probing disappears.

## Acceptance criteria

- Class-level encrypted-attribute checks resolve via `_encryptedAttributes`
  membership (Rails' surface) or the helper is removed; no caller depends on
  the pending/defs prototype walk.
- Mock-model tests that relied on the defs arm keep passing.
