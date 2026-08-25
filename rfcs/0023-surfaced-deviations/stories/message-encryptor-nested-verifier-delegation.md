---
title: "MessageEncryptor should delegate sign/verify to a nested MessageVerifier"
status: ready
updated: 2026-07-27
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activesupport"
deps: []
deps-rfc: []
est-loc: 150
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Rails' `MessageEncryptor` holds a nested `MessageVerifier` and delegates
signing to it: `sign(data)` is `@verifier.create_message(data)` and
`verify(data)` is `@verifier.read_message(data)`
(`vendor/rails/activesupport/lib/active_support/message_encryptor.rb:269-275`),
with `@verifier` built in the constructor and skipped entirely in AEAD mode.

trails' `packages/activesupport/src/message-encryptor.ts` instead HMACs inline:
`sign` is a direct `createHmac(...).digest("hex")` and `verify` a direct
`timingSafeEqual`, with no nested verifier. PR #5353 renamed the private
`verifySignature` to `verify` to match Rails' method name, which surfaced the
divergence in the wide call ratchet; it is baselined at
`scripts/api-compare/call-mismatches-wide-exclude/activesupport/message-encryptor.json`
(`verify` -> `read_message`) as a pre-existing structural divergence.

## Acceptance criteria

- Build the nested `@verifier` `MessageVerifier` in the constructor as Rails
  does, honoring the AEAD-mode skip.
- Route `sign` / `verify` through `createMessage` / `readMessage` on it.
- Delete the `verify` -> `read_message` entry from the wide-call exclude file
  rather than leaving it baselined.
- Existing message-encryptor / encrypted-file / credentials tests keep passing
  (the on-the-wire format must not change).

## Triage note (2026-08-18): the baseline path in this body is stale

This story cites `scripts/api-compare/call-mismatches-wide-exclude/…`. **That
tree no longer exists.** RFC 0084 folded the narrow RFC 0044 ratchet and the
wide one into a single gate over a single baseline:
`scripts/api-compare/call-mismatches-exclude/<package>/<tsFile .ts→.json>`,
gated by `pnpm parity:api:calls` (call-set rows) and `pnpm parity:api:calls:args`
(`kind: "args"` rows, RFC 0095).

Look for the row there, under the same `rubyName` / `call` pair. Everything else
in this story — the Rails and trails `file:line` citations, the described
divergence — is unaffected; only the path to the baseline row changed.

Remember the baseline is only-shrink: on converging, delete the one row by hand
(via `serializeBaseline`, sorted — never `--write`/reseed), then
`pnpm parity:api:calls:tighten <package>/<file>.json` for the stale high-water mark.
