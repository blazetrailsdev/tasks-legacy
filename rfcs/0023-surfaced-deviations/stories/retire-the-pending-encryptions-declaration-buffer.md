---
title: "Retire the trails-only _pendingEncryptions declaration buffer"
status: draft
updated: 2026-08-21
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 150
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Rails' `encrypt_attribute`
(`vendor/rails/activerecord/lib/active_record/encryption/encryptable_record.rb:84-95`)
keeps no per-class buffer of declarations. Its whole bookkeeping is
`encrypted_attributes << name.to_sym`, the `decorate_attributes` push, the
`preserve_original_encrypted` branch, and
`ActiveRecord::Encryption.encrypted_attribute_was_declared(self, name)`.

trails carries a second, parallel registry in
`packages/activerecord/src/encryption/encryptable-record.ts`:
`EncryptableRecord.registerPendingEncryption` writes a `PendingEncryption`
(`{ name, get scheme }`) onto `modelClass._pendingEncryptions`, which
`applyPendingEncryptions` (`packages/activerecord/src/encryption.ts:111`) reads
back to install the frozen-encryption validator, and `fixtures.ts`
(`encryptFixtureRows`) reads to encrypt fixture rows. `_pendingEncryptions` has
no Rails counterpart; the encrypted-attribute set plus `typeForAttribute` is the
only registry Rails has.

PR #6803 retired the other two consumers of the buffer (the idempotence guard
and `registerEncryptedType`); the buffer itself is what is left.

## Converged shape

Both remaining consumers read what Rails reads:

- The frozen-encryption validator installs off `encrypted_attributes` — Rails
  installs it via `validate :cant_modify_encrypted_attributes_when_frozen`
  (encryptable_record.rb:38), not off a declaration buffer.
- `encryptFixtureRows` resolves each attribute's scheme through
  `typeForAttribute(name)` (activemodel/lib/active_model/attribute_registration.rb:43-51),
  the same surface every other reader now uses since #6803.

`registerPendingEncryption`, the `PendingEncryption` interface, and
`_pendingEncryptions` are then deleted.

## Acceptance criteria

- `_pendingEncryptions` / `registerPendingEncryption` / `PendingEncryption` are
  gone from `encryptable-record.ts` and from every reader
  (`encryption.ts#applyPendingEncryptions`, `fixtures.ts`).
- `pnpm parity:api:extra --package activerecord` shows fewer novel names in
  `encryption/encryptable-record.ts`.
- The `packages/activerecord/src/encryption` suite stays green, including
  `encrypted-fixtures.test.ts` and the frozen-encryption cases in
  `encryptable-record.test.ts`.
