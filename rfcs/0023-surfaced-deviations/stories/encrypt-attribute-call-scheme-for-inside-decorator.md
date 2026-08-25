---
title: "encrypt_attribute must call scheme_for inside the decorator block, not via a lazy getter"
status: draft
updated: 2026-08-21
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 90
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced in PR #6806 while converging the encryption `columns_hash` readers. The
last remaining `encrypt_attribute` call-set row is
`order:constructor,schemeFor`, baselined at
`scripts/api-compare/call-mismatches-exclude/activerecord/encryption/encryptable-record.json`.
That baseline is debt, not permission.

`vendor/rails/activerecord/lib/active_record/encryption/encryptable_record.rb:84-96`:

```ruby
def encrypt_attribute(name, key_provider: nil, ...)
  encrypted_attributes << name.to_sym

  decorate_attributes([name]) do |name, cast_type|
    scheme = scheme_for key_provider: key_provider, key: key, deterministic: deterministic, ...

    ActiveRecord::Encryption::EncryptedAttributeType.new(scheme: scheme, cast_type: cast_type, default: columns_hash[name.to_s]&.default)
  end

  preserve_original_encrypted(name) if ignore_case
  ActiveRecord::Encryption.encrypted_attribute_was_declared(self, name)
end
```

Rails calls `scheme_for` **inside** the `decorate_attributes` block, on the line
before `EncryptedAttributeType.new`. trails builds a `PendingEncryption` struct
first and hangs `schemeFor` off a lazy `scheme` getter on it
(`packages/activerecord/src/encryption/encryptable-record.ts`, `encryptAttribute`

- `pushEncryptionDecorator`), so the analyzer records the constructor ahead of
  `schemeFor` and the two calls are inverted relative to Rails.

The getter exists for a real reason — "encryption schemes are resolved when
used, not when declared" (`encryptable_record_test.rb:388`), so a `configure`
between `encrypts` and first use must contribute its global previous schemes.
But Rails gets exactly that property for free by putting the `scheme_for` call
_in the block_, which is re-evaluated on every replay. The lazy getter is a
second mechanism for something the block already does.

## Converged shape

Call `schemeFor(options)` as the first statement of the decorator block passed
to `decorateAttributes`, and drop the `scheme` getter from `PendingEncryption`
(`_pendingEncryptions` would then carry only `name` plus whatever the
validator re-run path still needs). Replay already re-runs the block, so the
resolve-when-used semantics are preserved by construction.

Note `_pendingEncryptions` is read by `fixtures.ts` and
`applyPendingEncryptions`; check both before changing the struct's shape.

## Acceptance criteria

- [ ] `schemeFor` is called inside the decorator block, before the
      `EncryptedAttributeType` constructor.
- [ ] The `order:constructor,schemeFor` row is deleted from
      `call-mismatches-exclude/activerecord/encryption/encryptable-record.json`
      (no reseed; delete the single row, then `parity:api:calls:tighten` the
      stale mark shard).
- [ ] `encryptable_record_test.rb:388`'s "schemes are resolved when used, not
      when declared" behaviour still holds — the existing
      `encryption-schemes.test.ts` / `scheme.trails.test.ts` coverage stays green.
- [ ] Full `encryption/` suite green.
