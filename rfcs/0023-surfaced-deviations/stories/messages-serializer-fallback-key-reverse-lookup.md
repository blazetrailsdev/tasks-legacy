---
title: "Replace SerializerWithFallback key field with Rails' SERIALIZERS.key identity lookup"
status: ready
updated: 2026-07-27
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activesupport"
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

`packages/activesupport/src/messages/serializer-with-fallback.ts` (added by
PR #5351) carries a `key: Format` field on every serializer so `load` can put
the serializer's own name in the
`message_serializer_fallback.active_support` payload.

Rails has no such field: `SerializerWithFallback#load`
(`vendor/rails/activesupport/lib/active_support/messages/serializer_with_fallback.rb:22`)
recovers the name with `SERIALIZERS.key(self)` — a reverse lookup by object
identity. The field exists in trails only because the `*_allow_marshal`
variants inherit `format` from the serializer they wrap
(`serializer_with_fallback.rb:100-104`, `:139-143`), so `format()` alone cannot
distinguish `:json` from `:json_allow_marshal`.

## Acceptance criteria

- Replace the `key` field with an identity reverse lookup over `SERIALIZERS`
  (Rails' `SERIALIZERS.key(self)`), so the serializer objects carry only the
  members Rails' modules define.
- `messages/serializer_with_fallback.rb` stays at 8/8 in
  `pnpm parity:api --package activesupport`.
- The ported "notifies when serializer falls back to loading an alternate
  format" test still asserts `serializer: "marshal"` and continues to pass.
