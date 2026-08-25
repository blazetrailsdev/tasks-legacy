---
title: "EncryptedAttributeType#deserialize: drop the trails-only ?? fallback and optional call"
status: draft
updated: 2026-08-14
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 30
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced in PR #6540 (`naming-burndown-3-ar-model-encryption-tasks`) while
reading the last call-argument `naming` rows in the encryption cluster.

Rails `activerecord/lib/active_record/encryption/encrypted_attribute_type.rb:35-37`:

```ruby
def deserialize(value)
  cast_type.deserialize decrypt(value)
end
```

One expression: `decrypt` feeds `cast_type.deserialize` unconditionally.

trails `packages/activerecord/src/encryption/encrypted-attribute-type.ts:57-58`:

```ts
const decrypted = this.decrypt(value);
return this.castType.deserialize?.(decrypted) ?? decrypted;
```

Two deviations, and the second is why the local exists at all:

1. `castType.deserialize?.(...)` is optional-called, where Rails calls it
   unconditionally — `cast_type` is always a Type instance with `deserialize`.
2. The `?? decrypted` fallback returns the _decrypted ciphertext_ whenever the
   cast type's `deserialize` returns `null`/`undefined`. Rails returns that
   `nil` through. A cast type that legitimately deserializes to `nil` (a NULL
   column, `ActiveModel::Type::Value#deserialize` on a blank) therefore yields
   the raw decrypted string in trails and `nil` in Rails.

Because the local is read twice, it also stands where Rails inlines, which is
what the call-argument recorder flags (`deserialize` gets `ref:decrypted` vs
Rails' `ref:decrypt`).

## Converged shape

```ts
deserialize(value: unknown): unknown {
  return this.castType.deserialize(this.decrypt(value));
}
```

Dropping the optional call and the `??` fallback is a behavior change, so it
needs a test asserting a NULL-valued encrypted attribute deserializes to `null`
rather than to the decrypted text. Check whether any caller currently leans on
the fallback before removing it.

## Acceptance criteria

- [ ] `deserialize` is the single Rails expression, no `decrypted` local.
- [ ] `castType.deserialize` is called unconditionally.
- [ ] A test covers an encrypted attribute whose cast type deserializes to nil.
- [ ] The `encrypted-attribute-type.ts` `naming` call-arg row is gone.
