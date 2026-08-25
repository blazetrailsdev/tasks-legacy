---
title: "encrypt fixture rows through type_for_attribute, retiring the pending-encryptions walk"
status: draft
updated: 2026-08-21
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 80
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`encryptFixtureRows` in `packages/activerecord/src/fixtures.ts` still builds a
parallel `name → EncryptedAttributeType` map by walking the model's
`_pendingEncryptions` buffer and constructing a fresh `EncryptedAttributeType`
per pending declaration. PR #6807 converged the _cast type_ it wraps onto
`type_for_attribute`, but the wrapper is still hand-built from the pending
buffer rather than taken from the model's resolved type.

`_pendingEncryptions` is a trails invention; Rails encrypts fixture data by
serializing through the model's own declared type. The buffer exists here only
because fixtures can load before `loadSchemaFromAdapter` has run, at which point
`type_for_attribute` answers the fallback `Type.default_value`
(`activemodel/lib/active_model/attribute_registration.rb:37`) rather than the
decorated `EncryptedAttributeType` — the encryption decorator is queued but not
yet replayed.

## Converged shape

Serialize each encrypted fixture column through
`model_class.type_for_attribute(name)` alone, dropping the pending-buffer walk
and the local `EncryptedAttributeType` construction. That needs the decorator
replay to have happened by fixture-insert time — i.e. force the schema load
(`columns_hash` / `_default_attributes`) before encrypting, the way every other
resolved-type read in the file now does — instead of reconstructing the type to
work around a cold set.

## Acceptance criteria

- [ ] `encryptFixtureRows` reads only `typeForAttribute`; no `_pendingEncryptions`
      walk and no locally-constructed `EncryptedAttributeType`.
- [ ] The `defaultValue()` sentinel comparison introduced as the cold-set guard
      is gone with it.
- [ ] `fixtures` and `encryption/` suites pass, including encrypted-fixture
      round-trips.
