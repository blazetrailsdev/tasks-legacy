---
title: "Port an OpenSSL::Cipher-shaped object so new_cipher and key_len instantiate it"
status: draft
updated: 2026-08-12
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activesupport"
deps: []
deps-rfc: []
est-loc: 180
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`ActiveSupport::MessageEncryptor#new_cipher` is
`OpenSSL::Cipher.new(@cipher)` (`vendor/rails/activesupport/lib/active_support/message_encryptor.rb:367-369`).
trails' `newCipher` (`packages/activesupport/src/message-encryptor.ts`) instead
derives a spec object (`keyLen`/`ivLen`/`authenticated`) from the cipher NAME,
because there is no ported OpenSSL::Cipher. Two call-set baseline rows record
it: `key_len / new` (pre-existing) and `new_cipher / new` (added by PR #6431).

## Converged shape

A ported `OpenSSL::Cipher`-shaped object that `newCipher` instantiates and
`keyLen` reads, so both bodies read as Rails' do. Both baseline rows are then
deleted by hand (only-shrink).

## Acceptance criteria

- [ ] `newCipher` instantiates a cipher object rather than deriving a spec.
- [ ] `keyLen` reads that object's key length, matching message_encryptor.rb.
- [ ] Both `message-encryptor.json` `new` rows are deleted; encryption suites
      green on all three lanes.
