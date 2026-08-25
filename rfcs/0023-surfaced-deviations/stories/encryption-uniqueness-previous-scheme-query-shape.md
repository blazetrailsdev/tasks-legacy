---
title: "Converge encryption uniqueness previous-scheme query shape to Rails"
status: ready
updated: 2026-07-27
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 100
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`EncryptedUniquenessValidator#validateEach`
(`packages/activerecord/src/encryption/extended-deterministic-uniqueness-validator.ts:101-110`)
gates its previous-scheme uniqueness checks on `ExtendedDeterministicQueries.installed`,
and when it does run them it passes **all** previous ciphertexts as one array
rather than one scalar per type.

Rails does neither
(`vendor/rails/activerecord/lib/active_record/encryption/extended_deterministic_uniqueness_validator.rb:11-21`):

```ruby
encrypted_type.previous_types.each do |type|
  encrypted_value = type.serialize(value)
  ActiveRecord::Encryption.without_encryption do
    super(record, attribute, encrypted_value)
  end
end
```

There is no gate, and each `super` gets a scalar. The gate is also dead in any
real config: the railtie installs **both** patches under the same
`extend_queries` flag (`vendor/rails/activerecord/lib/active_record/railtie.rb:349-353`),
so `ExtendedDeterministicQueries.installed` is always true whenever the
validator patch is installed.

## Why the naive fix does not work

Removing the gate and iterating per-scalar (tried on #4988) turns
`errors.count` from 1 into **2** in three tests, including the Rails-named
"uniqueness validation does not revalidate the attribute with current
encryption type" (`uniqueness_validations_test.rb:59`).

Root cause: the plaintext variant gets matched **twice**.

- Rails finds it once, via the clean-text scheme that `previous_types`
  appends when `support_unencrypted_data?`
  (`encrypted_attribute_type.rb:66-68`). trails mirrors this correctly
  (`encrypted-attribute-type.ts:209-212`).
- trails _also_ finds it in the **first** `super` call, because
  `buildRelation` has a `supportUnencryptedData` special-case that switches to
  a hash-style `where` so `processArguments` expands the IN list to include
  the plain-text variant (`packages/activerecord/src/validations/uniqueness.ts:415-423`).

That `buildRelation` special-case is the actual trails invention. The
`ExtendedDeterministicQueries.installed` gate is a band-aid compensating for
the resulting double-match.

## Acceptance criteria

- Remove the `supportUnencryptedData` hash-style-`where` special-case from
  `buildRelation` in `validations/uniqueness.ts` so the first `super` queries
  only the current ciphertext, as in Rails.
- Remove the `ExtendedDeterministicQueries.installed` gate and iterate
  `previousTypes` calling `originalValidateEach` once per **scalar**
  `previousType.serialize(value)`, inside `withoutEncryption` — mirroring
  `extended_deterministic_uniqueness_validator.rb:15-21`.
- Update the `EncryptedUniquenessValidator` unit test that asserts the batched
  array shape (`extended-deterministic-uniqueness-validator.test.ts:56-58`) to
  expect one call per previous type. Do NOT rename the test.
- All six tests in `encryption/uniqueness-validations.test.ts` stay green, with
  `errors.count` of 1 (not 2) in the mixed encrypted/unencrypted cases.

## Notes

Identified by a Codex review on #4988. That PR keeps the gate (it is green) and
only fixed a dropped-promise bug in the same file; converging the query shape
is a separate multi-file change in the encryption query stack.
