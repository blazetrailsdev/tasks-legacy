---
title: "MessagePackMessageSerializer#load converts to Buffer at the call site instead of at the byte/string boundary"
status: draft
updated: 2026-08-23
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
  - "activesupport"
deps: []
deps-rfc: []
est-loc: 80
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`packages/activerecord/src/encryption/message-pack-message-serializer.ts:27`
`load` calls
`MessagePack.load(Buffer.from(serializedContent, "latin1"))` where Rails calls
`ActiveSupport::MessagePack.load(serialized_content)` directly
(`vendor/rails/activerecord/lib/active_record/encryption/message_pack_message_serializer.rb:27`).

The conversion exists because Ruby's `serialized_content` is a binary String
while `MessagePack.load` in trails is typed `load(dumped: Buffer)`
(`packages/activesupport/src/message-pack/serializer.ts:42`) — the serializer
interface carries the packed bytes as a latin1 string (one char per byte) so the
binary form survives a string-typed `MessageSerializerLike`. So the divergence is
the Ruby-String-is-bytes boundary, resolved at the call site instead of in the
type.

Surfaced by RFC 0096 wave-5 (`wave-5-naming-tail`) as a `naming` call-arg row —
ruby `ref:serializedContent` vs ts `ref:from` — but it is an a3 (invented
conversion) finding, not a rename, and closing it would change
`ActiveSupport::MessagePack.load`'s public signature, which wave-5 forbids. It
was therefore re-filed rather than renamed away, and the row stands in
`scripts/api-compare/output/call-arg-mismatches.json` until this lands.

## Acceptance criteria

- Either `MessagePack.load` accepts the byte-carrying string trails' encryption
  serializers hold (so the call site passes `serializedContent` the way Rails
  passes `serialized_content`), or the byte/string boundary is moved to the
  `MessageSerializerLike` interface so no per-call-site conversion is needed.
- The `activerecord/encryption/message-pack-message-serializer.ts load` row
  disappears from `pnpm parity:api:calls:args:report` with no baseline row added.
- `dump`'s matching `.toString("latin1")` hop is addressed by the same decision.
