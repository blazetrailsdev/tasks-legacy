---
title: "AdapterNotFound message is built twice: resolve and the sync validateAdapterName twin"
status: draft
updated: 2026-08-25
rfc: "0119-connection-adapter-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 60
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while landing `adapter-not-found-message-should-be-built-inline`
(PR #7046), which inlined the `AdapterNotFound` message into `resolve` as Rails
builds it (`vendor/rails/activerecord/lib/active_record/connection_adapters.rb:34-39`)
and deleted the private `adapterNotFoundError` helper Rails does not have.

That left the message text duplicated. `packages/activerecord/src/connection-adapters.ts`
has a second raise site, `validateAdapterName`, a trails-only synchronous twin
of `resolve` (`@internal`, no Ruby counterpart) that now repeats the same four
interpolated lines because the helper it shared is gone. Rails has ONE builder,
inside `resolve`.

`validateAdapterName` exists because trails' sync callers cannot await the
dynamic import: it is reached from
`connection-adapters/abstract/connection-handler.ts:36` (registered as a
validator) and called through `database-configurations/database-config.ts:354-359`
(`_validateAdapterName`) on `dbConfig.newConnection`, which is synchronous as
Rails' is. In Ruby the same check is inline in `resolve` because `require` is
synchronous — this is the async/sync split, not a design choice.

Note the file already carries `resolveSync` / `resolveSyncError` for the same
reason, so the convergence question is the whole sync-companion cluster, not
just this one function.

## Acceptance criteria

- [ ] The AdapterNotFound message is built in exactly one place, as Rails
      builds it once in `resolve` (connection_adapters.rb:34-39), with the
      sorted `[...adapters.keys()].sort().join(", ")` intact.
- [ ] `validateAdapterName` either disappears (its callers reaching the async
      `resolve`, if the call chain can be made async) or its remaining
      existence is justified at the call site as the sync-companion language
      shortcoming it is — NOT by reintroducing a shared private helper Rails
      has no counterpart for, which is exactly what PR #7046 removed.
- [ ] `pnpm parity:api:extra --package activerecord` does not grow.
- [ ] `pnpm parity:api:calls` stays clean: the `join` row for
      `connection-adapters.json` was deleted by PR #7046 and must not come back.
