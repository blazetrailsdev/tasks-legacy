---
title: "EncryptedFile#encryptor passes NullSerializer where Rails passes Marshal"
status: ready
updated: 2026-08-22
rfc: "0112-one-rails-thing-n-trails-things"
cluster: duplicate-bodies
deps: []
deps-rfc: []
est-loc: 120
priority: 50
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`packages/activesupport/src/encrypted-file.ts` builds its encryptor with
`serializer: NullSerializer`. Rails builds it with `serializer: Marshal`:

```ruby
# vendor/rails/activesupport/lib/active_support/encrypted_file.rb:113
def encryptor
  @encryptor ||= ActiveSupport::MessageEncryptor.new([ key ].pack("H*"), cipher: CIPHER, serializer: Marshal)
end
```

Surfaced while converging `CIPHER` to `aes-128-gcm` (PR #5994, story
`converge-encrypted-file-cipher-to-aes-128-gcm`). The cipher was the scoped
change; the serializer is an independent deviation on the adjacent line and was
left in place, recorded in the file header's documented-divergences JSDoc.

Consequence: the on-disk payload format differs from Rails'. A trails-written
`.enc` file holds the raw plaintext string inside the envelope where Rails holds
a Marshal dump, so the two implementations cannot read each other's files even
with a matching key and cipher. Callers compensate at a higher level —
`EncryptedConfiguration` parses the contents itself rather than relying on the
serializer to round-trip a structured value.

Likely shares a blocker with the draft story
`messages-codec-default-serializer-marshal` (same missing Marshal port, at the
`Codec` default-serializer level rather than this call site); triage the two
together.

## Converged shape

`encryptor()` passes a `Marshal` serializer, matching `encrypted_file.rb:113`,
and the `NullSerializer` bullet is removed from the `encrypted-file.ts` header
JSDoc. Requires a Marshal port (or a decision on the minimum Marshal subset
`EncryptedFile` needs — Rails only ever round-trips a String here).

## Acceptance criteria

- `encryptor()` in `packages/activesupport/src/encrypted-file.ts` passes the
  Rails serializer, cited to `encrypted_file.rb:113`.
- The `NullSerializer` divergence bullet is deleted from the file header.
- `encrypted-file.test.ts` and `encrypted-configuration.test.ts` still pass;
  `EncryptedConfiguration`'s own parsing is reconciled with whatever the
  serializer now returns.
- If a Marshal port is genuinely out of reach, `pnpm tasks block` with the
  specific blocker rather than re-justifying the deviation.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
