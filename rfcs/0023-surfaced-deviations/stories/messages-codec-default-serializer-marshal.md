---
title: "Codec default_serializer is :json in trails, :marshal in Rails"
status: ready
updated: 2026-07-27
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activesupport"
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

Rails' `Messages::Codec` declares
`class_attribute :default_serializer, default: :marshal`
(`vendor/rails/activesupport/lib/active_support/messages/codec.rb:12-13`) and
neither `MessageEncryptor` nor `MessageVerifier` overrides it.

PR #5353 deviated: both subclasses set
`static override defaultSerializer = "json"`
(`packages/activesupport/src/message-encryptor.ts`,
`packages/activesupport/src/message-verifier.ts`). Justification at the time:
every signed id, SGID, and encrypted credentials file already written by trails
is a JSON payload, so flipping the default would make existing messages
unreadable.

Converging needs a migration story, not just a constant flip — most likely
leaning on `SerializerWithFallback`'s fallback detection (`:marshal` already
falls back to reading JSON) plus a `Rails.application.config` style opt-in,
mirroring how Rails shipped `active_support.message_serializer`.

## Acceptance criteria

- Decide and document whether trails can adopt Rails' `:marshal` default.
- If yes: flip both subclasses to inherit `Codec`'s default and confirm
  existing JSON messages still verify via the with-fallback path.
- If no: record the deviation in the surfaced-deviations ledger with the
  reasoning, and delete the two `defaultSerializer` overrides in favor of a
  single documented configuration point.
