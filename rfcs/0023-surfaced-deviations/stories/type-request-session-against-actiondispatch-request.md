---
title: "Drop Request::Session's structural Req stand-in for the real Request"
status: draft
updated: 2026-08-18
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "actionpack"
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

`packages/actionpack/src/action-dispatch/request/session.ts` declares its own
structural stand-in for the request rather than using the ported
`ActionDispatch::Request`:

```ts
/**
 * @noRailsEquivalent PERMANENT — structural stand-in for `ActionDispatch::Request`,
 * which this file must not import (it would close a module cycle).
 */
export type Req = { env: Record<string, unknown> };
```

Rails passes the real request everywhere (`request/session.rb:19`, `:35-45`,
`:65-69`, `:97-108`) and calls `req.get_header` / `req.set_header` /
`req.delete_header` on it; the port indexes `req.env` directly because the
structural type exposes nothing else. `http/request.ts` imports `Session`, so a
back-import closes a cycle — the reason the stand-in exists — but the
`PERMANENT` claim was asserted, not proven: the settled trails answer for an
import that closes a cycle is a zero-import slot module (see CLAUDE.md,
"Call-time constant resolution"), and it was never tried here.

The same stand-in makes `Session` skip `get_header`/`set_header`/`delete_header`
entirely, which is a second, downstream deviation: those are ported methods on
`Request` that the gate cannot see us omitting because the receiver is typed as
a bare `{ env }`.

## Converged shape

Type `by`/`req` against the real `ActionDispatch::Request` (via a type-only
import, or the slot idiom if a runtime edge is genuinely needed), then spell the
header access as Rails does — `req.getHeader(ENV_SESSION_KEY)`,
`req.setHeader(...)`, `req.deleteHeader(...)` — and drop the `Req` alias and its
`@noRailsEquivalent` tag. Verify the load order against BUILT `dist/**.js` with
plain node, not vitest (a vitest run enters the funnel module first and masks a
TDZ).

## Acceptance criteria

- [ ] `Req` and its `@noRailsEquivalent` tag are gone.
- [ ] `Session` / `Session::Options` reach the request through the ported
      header methods, matching `request/session.rb`.
- [ ] No import cycle: a plain-node import of the built `dist` modules as entry
      modules succeeds from both `request.js` and `session.js`.
