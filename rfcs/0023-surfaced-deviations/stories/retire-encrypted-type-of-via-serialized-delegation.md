---
title: "Retire encryptedTypeOf by giving Serialized/NormalizedValueType DelegateClass forwarding"
status: draft
updated: 2026-08-21
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 160
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced in PR #6806. `encryptedTypeOf` is the only remaining novel name in
`packages/activerecord/src/encryption/encryptable-record.ts` under
`pnpm parity:api:extra --package activerecord` — a trails invention with no Ruby
counterpart:

```ts
export function encryptedTypeOf(type: unknown): EncryptedAttributeType | undefined {
  let current: any = type;
  while (current) {
    if (current instanceof EncryptedAttributeType) return current;
    current = current.subtype ?? current.castType;
  }
  return undefined;
}
```

It exists because Rails gets the same reach for free from `DelegateClass`.
`vendor/rails/activerecord/lib/active_record/type/serialized.rb` is
`class Serialized < DelegateClass(ActiveModel::Type::Value)`, and
`vendor/rails/activerecord/lib/active_record/normalized_value_type.rb` likewise
delegates, so a wrapped `Serialized(Encrypted(binary))` answers `encrypted?`,
`deterministic?` and `previous_types` by forwarding to the wrapped type. Rails
therefore just writes `type.previous_types` / `type.deterministic?` on whatever
`type_for_attribute` returned — see
`vendor/rails/activerecord/lib/active_record/encryption/encryptable_record.rb:145-153`
(`encrypted_attribute?`) and
`vendor/rails/activerecord/lib/active_record/encryption/extended_deterministic_queries.rb`.

trails has no `DelegateClass`, so every call site that wants the encrypted type
hand-unwraps through this helper. Current callers:
`encryptable-record.ts` (`encryptedAttribute`, `deterministicEncryptedAttributes`),
`extended-deterministic-queries.ts`, `extended-deterministic-uniqueness-validator.ts`,
and `encryption.ts`.

## Converged shape

Give the wrapping types the delegation Ruby's `DelegateClass` supplies, so the
predicates answer through the wrapper and the call sites read like Rails:

- `Type::Serialized` and `NormalizedValueType` forward the encryption-facing
  members (`isEncrypted`, `deterministic`, `previousTypes`, `castType`) to their
  wrapped subtype, matching `serialized.rb`'s `DelegateClass(...)` and
  `normalized_value_type.rb`.
- Call sites drop `encryptedTypeOf(...)` and read the member directly off
  `typeForAttribute(name)`, as Rails does.
- Delete `encryptedTypeOf` and its `@noRailsEquivalent`-shaped surface; confirm
  `parity:api:extra --package activerecord` shows `encryptable-record.ts` at 0
  novel.

Check whether a generic forwarding shape already exists in the repo before
hand-rolling per-type delegation — RFC 0058
(`delegation-remaining-delegate-class-prototype-carriers`, done) established the
prototype-carrier idiom for `DelegateClass` ports and is the precedent to follow.

## Acceptance criteria

- [ ] `encryptedTypeOf` is deleted; no call site hand-unwraps a type.
- [ ] `Serialized` / `NormalizedValueType` answer the encryption predicates by
      delegation, per `serialized.rb` and `normalized_value_type.rb`.
- [ ] `pnpm parity:api:extra --package activerecord` reports 0 novel names for
      `encryption/encryptable-record.ts`.
- [ ] `pnpm parity:api:calls` / `:args` non-negative; no new baseline rows.
- [ ] Full `encryption/` suite green, including
      `extended-deterministic-queries`, the uniqueness validator, and the
      serialized/normalized encryption tests.
