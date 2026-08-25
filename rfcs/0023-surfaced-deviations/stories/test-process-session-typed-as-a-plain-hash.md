---
title: "TestProcess#session is typed Record<string, unknown>, not Session"
status: draft
updated: 2026-08-18
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "actionpack"
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

`ActionDispatch::TestProcess#session`
(`vendor/rails/actionpack/lib/action_dispatch/testing/test_process.rb`) is
`@request.session` — it answers whatever `Request#session` answers, which since
PR #6696 is a real `ActionDispatch::Request::Session`
(`vendor/rack/lib/rack/request.rb:207-211` +
`vendor/rails/actionpack/lib/action_dispatch/http/request.rb:505-507`).

trails' port still types it as a plain hash:

- `packages/actionpack/src/action-dispatch/testing/test-process.ts:24` —
  `TestProcessHost`'s request declares `session: Record<string, unknown>`.
- `test-process.ts:125-126` — `session()` is annotated
  `: Record<string, unknown>`.

The mismatch is invisible to `tsc` only because `TestProcessHost` is a
structural interface distinct from the real `Request` class, so the two
`session` types never meet. A caller that reaches for `Session` API
(`isEnabled()`, `hasKey()`, `destroy()`) through the test helper gets a type
error against a value that in fact supports it.

Sibling shapes to check in the same pass:
`packages/actionpack/src/action-controller/metal/request-forgery-protection.ts:540-542`
casts `c.session` to `Record<string, unknown>` for the same reason.

## Acceptance criteria

- [ ] `TestProcessHost.request.session` and `session()`'s return type are
      `Session`, matching what `Request#session` now answers.
- [ ] The `Record<string, unknown>` casts on `controller.session` in
      `request-forgery-protection.ts` are dropped or narrowed to the `Session`
      API Rails actually calls.
- [ ] `pnpm typecheck --force` clean; actionpack tests green.
