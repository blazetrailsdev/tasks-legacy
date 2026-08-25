---
title: "get-crypto-sync-auto-registration-has-no-esm-arm"
status: ready
updated: 2026-08-25
rfc: "0113-branch-and-guard-parity"
cluster: missing-arm
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

# `getCrypto()`'s sync auto-registration has no ESM arm

## Context

`packages/activesupport/src/crypto-adapter.ts:169-192` — `tryAutoRegisterNode()`
reaches `node:crypto` through `createRequire`, but only when
`typeof require !== "undefined"` (:178-181). Under pure ESM (`tsx`, a plain
`node dist/**.js` entry) that is false, so the function returns `false` and
`resolve()` (:194-213) throws
`No crypto adapter configured. Set ActiveSupport.cryptoAdapter or register a
custom adapter.` Only the async arm — `tryAutoRegisterNodeAsync()` (:219-238),
a dynamic `import("node:crypto")` — works there, and it has to have been
awaited by someone first.

Surfaced in PR #6561: `Instrumenter#unique_id` (`SecureRandom.hex(10)`,
`notifications/instrumenter.rb:100-102`) is the first sync `getCrypto()` caller
on the instrumentation path, which runs before any host warms the adapter.
`scripts/parity/pipeline/query/node/ar_dump.ts` swallowed the throw as
`schema pre-warm failed for User: No crypto adapter configured`, which left
`columnsHash()` cold and dropped the ORDER BY table qualifier in fixture
`ar-12`. That PR unblocked itself with a Web Crypto fallback in `unique_id`
alone; every other sync `getCrypto()` caller (`digest.ts`,
`message-encryptor.ts`, `security-utils.ts`, `secure-password.ts`,
`core-ext/securerandom.ts`, `digest/uuid.ts`, …) still throws in the same
window.

The honest fix is at the seam, not per-caller. Note that a Web-Crypto-backed
adapter cannot be registered as a general answer: it can serve `randomBytes`
and `randomUUID` but not `createHash`/`createHmac`/`createCipheriv`, so
registering a partial adapter would turn a clear "not configured" error into
`createHash is not a function`.

## Acceptance criteria

- [ ] `getCrypto()` resolves the Node adapter under pure ESM without a prior
      `getCryptoAsync()` await, or the seam documents precisely why it cannot
      and what callers must do instead.
- [ ] A regression test runs a sync `getCrypto()` caller as an ESM entry
      module (not through vitest, which enters via CJS interop and masks it).
- [ ] `Instrumenter#unique_id` drops its Web Crypto fallback and goes back to
      the seam unconditionally.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0113-branch-and-guard-parity`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
