---
title: 'parse_properties routes a nested hash by is_a?(Hash), not by a "p" probe'
status: draft
updated: 2026-08-14
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

Surfaced while converging `hash_to_message` in PR #6493.

`packages/activerecord/src/encryption/message-serializer.ts:85`
(`parseProperties`) decides whether a header value is a nested message with a
`"p" in value` probe:

```ts
typeof value === "object" && value !== null && !Array.isArray(value) && "p" in value
  ? this.parseMessage(value, level + 1)
  : this.decodeIfNeeded(value);
```

Rails asks only whether the value is a Hash
(`vendor/rails/activerecord/lib/active_record/encryption/message_serializer.rb:57`):

```ruby
properties[key] = value.is_a?(Hash) ? parse_message(value, level + 1) : decode_if_needed(value)
```

The difference is observable: a headers hash WITHOUT a `"p"` key routes to
`decode_if_needed` in the port and to `parse_message` in Rails, where
`validate_message_data_format` raises
`Errors::Decryption, "Invalid data format: hash without payload"`
(`message_serializer.rb:50`). The port silently returns the hash instead of
raising. The sibling `message-pack-message-serializer.ts` already ports the
plain object arm without the `"p"` probe, so the two serializers disagree.

## Converged shape

`parseProperties` routes every plain-object value to `parseMessage`, letting
`validateMessageDataFormat` raise as Rails does. Cover the raise with a test
mirroring `message_serializer_test.rb`'s invalid-format cases.

## Acceptance criteria

- [ ] `message-serializer.ts#parseProperties` branches on "is a plain object",
      not on `"p" in value` (`message_serializer.rb:57`).
- [ ] A header whose value is an object without `"p"` raises `Decryption`
      ("Invalid data format: hash without payload"), as `message_serializer.rb:50`.
- [ ] Encryption suites green.
