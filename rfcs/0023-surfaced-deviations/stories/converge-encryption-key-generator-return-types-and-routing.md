---
title: "Converge encryption KeyGenerator return types and generateRandomHexKey routing"
status: draft
updated: 2026-07-27
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 120
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Found by the `activerecord-unrouted-privates-drop-carried-arguments` sweep
(PR #5419) while triaging unrouted privates that carry arguments.

`packages/activerecord/src/encryption/key-generator.ts` diverges from
`activerecord/lib/active_record/encryption/key_generator.rb` in two coupled
ways, which is why the sweep deferred it rather than routing it:

**1. Return type.** Rails' `generate_random_key(length:)` returns
`SecureRandom.random_bytes(length)` — raw bytes. trails returns
`.toString("base64")` (key-generator.ts:28-32), so a 32-byte key surfaces as a
44-char base64 string. `derive_key_from` has the same shape of deviation.

**2. Dropped routing.** Rails' `generate_random_hex_key` is defined _in terms
of_ the former:

```ruby
def generate_random_hex_key(length: key_length)
  generate_random_key(length: length).unpack("H*")[0]
end
```

trails re-issues `getCrypto().randomBytes(...)` independently
(key-generator.ts:34-38), so `generateRandomKey` is unrouted and an override of
it is bypassed.

The routing cannot be fixed without settling (1) first: routing hex through a
base64-returning `generateRandomKey` requires decoding back to bytes, which is
strictly worse than the current duplication.

## Also noted

`deriveKey(password, length, salt)` (key-generator.ts:40-47) has no counterpart
in Rails' KeyGenerator at all — confirm whether it is allowlisted extra surface
or should be retired as part of this work.

## Acceptance criteria

- `generateRandomKey` / `deriveKeyFrom` return types reconciled with Rails, or
  the deviation justified at the call site if callers depend on base64.
- `generateRandomHexKey` routes through `generateRandomKey` once the return
  type is settled.
- Test asserting the routing (an overridden `generateRandomKey` is observed by
  `generateRandomHexKey`), verified to FAIL beforehand.
- Encryption round-trip tests stay green — key material shape is load-bearing.
- `deriveKey`'s status resolved explicitly.
