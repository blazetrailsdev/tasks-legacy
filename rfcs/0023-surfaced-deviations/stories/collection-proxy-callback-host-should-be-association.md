---
title: "CollectionProxy should delegate callback dispatch to CollectionAssociation instead of a fabricated host"
status: ready
updated: 2026-07-27
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 90
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`CollectionAssociation#callback` / `callbacksFor`
(`packages/activerecord/src/associations/collection-association.ts`, Rails
`collection_association.rb:492` and `:498`) are module-level functions taking a
structural `CallbackHost { owner, reflection }` rather than instance methods on
`CollectionAssociation`. That shape exists only because `CollectionProxy` does
not hold a `CollectionAssociation` — it holds the same two pieces under its own
field names (`_record`, `_assocDef`) and fabricates a host object via the
`_callbackHost` getter (`associations/collection-proxy.ts`).

In Rails the proxy delegates to the association (`CollectionProxy` holds
`@association` and forwards), so `callback` is a plain private instance method
and there is no host interface at all. Introduced by PR #5365, which routed the
proxy's add/remove paths through the Rails-named `callback` and deleted the
duplicate `fireAssocCallbacks` dispatcher in `associations.ts`.

This is the same root cause as the wider "proxy and OO association have
separate targets" divergence — the proxy is not a thin delegator over the
association.

## Acceptance criteria

- `CollectionProxy` reaches its `CollectionAssociation` (or the association
  reaches the proxy) so callback dispatch does not need a fabricated host.
- `callback` and `callbacksFor` become instance methods on
  `CollectionAssociation`, matching Rails' private instance methods.
- The `CallbackHost` interface and the `_callbackHost` getter are deleted.
- `pnpm parity:api` shows no new extras; `associations/collection-association.ts`
  novel count does not rise above 4.
- `associations/callbacks.test.ts`, `associations/callbacks.trails.test.ts`,
  `associations/collection-proxy.test.ts` and the HABTM/nested-attributes
  callback suites pass unchanged. No test renames.
