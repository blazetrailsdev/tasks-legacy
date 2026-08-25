---
title: "Derive Aes256Gcm's key/iv lengths from the cipher and drop the invented _validateKeyLength"
status: draft
updated: 2026-08-13
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 70
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while converging `Aes256Gcm#encrypt`/`#decrypt` in PR #6483.

Rails' `Aes256Gcm` derives both lengths from the cipher itself, as class
methods (`vendor/rails/activerecord/lib/active_record/encryption/cipher/aes256_gcm.rb:17-25`):

```ruby
CIPHER_TYPE = "aes-256-gcm"

class << self
  def key_length
    OpenSSL::Cipher.new(CIPHER_TYPE).key_len
  end

  def iv_length
    OpenSSL::Cipher.new(CIPHER_TYPE).iv_len
  end
end
```

trails hard-codes them as module constants and exposes them as static
fields (`packages/activerecord/src/encryption/cipher/aes256-gcm.ts:11-13,
:32-33`):

```ts
const KEY_LENGTH = 32;
const IV_LENGTH = 12;
static keyLength = KEY_LENGTH;
static ivLength = IV_LENGTH;
```

Separately, the port carries a private `_validateKeyLength(key)`
(`:141-148`) with no Rails counterpart. Rails does no such pre-check: it
assigns `cipher.key = @secret` (`:41`) and lets OpenSSL raise on a
short key. The trails helper also raises `Encryption::Errors::Configuration`
with a bespoke message (`"The provided key has length N but must be at
least 32 bytes"`), which is neither Rails' error class nor Rails' string for
that failure.

## Converged shape

1. Derive `keyLength` / `ivLength` from the cipher rather than from
   hard-coded constants, so a `CIPHER_TYPE` change cannot desync them —
   node's `crypto.getCipherInfo("aes-256-gcm")` returns `{ keyLength,
ivLength }`, which is the direct analogue of `key_len` / `iv_len`.
   Keep them class-level, as Rails does.
2. Delete `_validateKeyLength` and let `createCipheriv` raise, matching
   Rails' raise site (`:41`). Check what the resulting error looks like
   through `Encryptor`'s per-key retry loop before landing — if the bare
   node error must be translated, translate it at the same site Rails'
   OpenSSL error surfaces from, not in a pre-check.

## Acceptance criteria

- [ ] `keyLength` / `ivLength` are derived from `CIPHER_TYPE`, not literals.
- [ ] `_validateKeyLength` is gone; a short key raises where Rails raises.
- [ ] `pnpm parity:api:extra --package activerecord` shows no new surface.
- [ ] Encryption suites pass on all three adapters.
