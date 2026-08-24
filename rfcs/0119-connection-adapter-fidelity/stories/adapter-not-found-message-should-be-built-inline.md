---
title: "adapter-not-found-message-should-be-built-inline"
status: draft
updated: 2026-08-22
rfc: "0119-connection-adapter-fidelity"
cluster: null
packages: []
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

Rails `vendor/rails/activerecord/lib/active_record/connection_adapters.rb:34-39`
builds the `AdapterNotFound` message **inline** inside `resolve`. trails
`packages/activerecord/src/connection-adapters.ts` extracts it into a private
`adapterNotFoundError` helper Rails does not have, so `resolve`'s call set no
longer contains the `join`.

CLAUDE.md: "If Rails inlines something, inline it. One Rails method is one TS
method." The extraction is exactly the kind of abstraction Rails does not have.

Surfaced by the RFC 0106 call-set gate (the `join` row on
`connection-adapters.json`, now a CONVERGEABLE `@missingRailsCall` tag).

## Acceptance criteria

- [ ] `resolve` builds the `AdapterNotFound` message inline, matching connection_adapters.rb:34-39 line for line, including the `[...adapters.keys()].sort().join(", ")`.
- [ ] The private `adapterNotFoundError` helper is deleted.
- [ ] The `@missingRailsCall join` tag on `connection-adapters.ts` is deleted, not re-justified.
- [ ] `pnpm parity:api:extra --package activerecord` shows one fewer extra name.
