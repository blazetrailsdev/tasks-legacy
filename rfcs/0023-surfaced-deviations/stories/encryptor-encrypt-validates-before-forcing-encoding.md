---
title: "Encryptor#encrypt validates the payload before forcing the encoding; Rails forces first"
status: draft
updated: 2026-08-23
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

`packages/activerecord/src/encryption/encryptor.ts:132-133` `Encryptor#encrypt`
runs Rails' first two statements in the opposite order. Rails
(`vendor/rails/activerecord/lib/active_record/encryption/encryptor.rb:49-52`):

```ruby
def encrypt(clear_text, key_provider: default_key_provider, cipher_options: {})
  clear_text = force_encoding_if_needed(clear_text) if cipher_options[:deterministic]

  validate_payload_type(clear_text)
  serialize_message build_encrypted_message(clear_text, ...)
end
```

trails validates first and forces after:

```ts
this.validatePayloadType(clearText);
if (options?.deterministic) clearText = this.forceEncodingIfNeeded(clearText);
```

Rails validates the _forced_ value; trails validates the raw one. The two agree
whenever `forced_encoding_for_deterministic_encryption` is unset (trails'
`forceEncodingIfNeeded` returns the value unchanged), which is why no test sees
it today, but the branch order is a real divergence and the error a non-String
deterministic payload raises differs between the two.

Spotted while landing RFC 0096's `wave-5-naming-tail`, which converged the
`clear_text` identifier in this method but is explicitly forbidden from changing
behaviour or control flow, so the swap was filed instead of taken there.

## Acceptance criteria

- `encrypt` runs `forceEncodingIfNeeded` before `validatePayloadType`, matching
  `encryptor.rb:49-52`.
- The non-String-payload arms are covered for both the deterministic and
  non-deterministic paths, so the reordering is pinned rather than assumed inert.
