---
title: "override_accessors_to_preserve_original is a beforeSave hook, not an included module"
status: draft
updated: 2026-08-18
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 140
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

# override_accessors_to_preserve_original is a beforeSave hook, not an included module

## Context

Surfaced in PR #6719 (RFC 0106 wave 4c, encryption slice) while reasoning the
`override_accessors_to_preserve_original` call-set rows — the two rows
(`include`, `encrypted_attribute?`) are baselined at
`scripts/api-compare/call-mismatches-exclude/activerecord/encryption/encryptable-record.json`
and that baseline is debt, not permission.

`vendor/rails/activerecord/lib/active_record/encryption/encryptable_record.rb:109-124`:

```ruby
def override_accessors_to_preserve_original(name, original_attribute_name)
  include(Module.new do
    define_method name do
      if ((value = super()) && encrypted_attribute?(name)) || !ActiveRecord::Encryption.config.support_unencrypted_data
        send(original_attribute_name)
      else
        value
      end
    end

    define_method "#{name}=" do |value|
      self.send "#{original_attribute_name}=", value
      super(value)
    end
  end)
end
```

`packages/activerecord/src/encryption/encryptable-record.ts:286-321` does
something structurally different:

- a `beforeSave` callback that copies `name` into `original_<name>` when the
  attribute is dirty (Rails has no callback here at all — the writer does the
  copy inline, every time), plus
- an `Object.defineProperty` on the prototype whose **reader** is
  `readAttribute(original) ?? readAttribute(name)`.

Rails' reader condition is `(super() && encrypted_attribute?(name)) || !config.support_unencrypted_data`
— it prefers the original **only when the current value is actually encrypted**,
or when unencrypted data is not supported at all. trails prefers the original
whenever it is non-null, which is a different predicate: a row holding
plaintext under `support_unencrypted_data` returns the preserved original where
Rails returns the plaintext it just read. The dirty-gated callback is also
observably different from an unconditional writer — a value assigned and then
read back before save takes a different path.

## Converged shape

The blocker is real but narrow: TS mixins cannot `super()` into the ancestor
chain (CLAUDE.md "Module mixins"). The settled workaround is to capture the
prior descriptor and call it explicitly, which is what the `super()` in both
generated methods resolves to:

- capture `Object.getOwnPropertyDescriptor(proto, name)` (falling back to the
  attribute-methods accessor) before redefining — that captured getter/setter IS
  `super`;
- reader: `const value = super(); if ((value != null && value !== false && encryptedAttribute.call(this, name)) || !Configurable.config.supportUnencryptedData) return this[originalAttributeName]; return value;`
  — note Ruby truthiness on `value`, not `!= null`;
- writer: `this[originalAttributeName] = value; super(value);` — unconditional,
  in that order;
- delete the `beforeSave` hook and the dirty gate entirely; Rails has neither.

Then delete the two `override_accessors_to_preserve_original` rows from the
baseline by hand via `serializeBaseline` and
`pnpm parity:api:calls:tighten activerecord/encryption/encryptable-record.json`.

## Acceptance criteria

- [ ] `overrideAccessorsToPreserveOriginal` reproduces both generated methods'
      bodies from `encryptable_record.rb:111-122`, including the reader's full
      three-term condition and Ruby truthiness on `super()`'s value.
- [ ] No `beforeSave` callback and no dirty gate remain in that method.
- [ ] The two baseline rows are deleted (not re-reasoned) and the shard
      tightened. No `--write`, no reseed.
- [ ] A test covers the arm the current predicate gets wrong: `ignore_case` with
      `support_unencrypted_data` on and a plaintext value present, asserting the
      reader returns the plaintext rather than the preserved original.
- [ ] Rails test names verbatim; canonical schema and models only.
- [ ] `pnpm parity:api:calls` green; SQLite, PostgreSQL and MySQL/MariaDB lanes
      green.
