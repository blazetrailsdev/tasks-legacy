---
title: "mysql2-connected-drops-invented-permanently-closed-and-fake-terms"
status: ready
updated: 2026-08-22
rfc: "0112-one-rails-thing-n-trails-things"
cluster: split-stores
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

`Mysql2Adapter#isConnected` (`packages/activerecord/src/connection-adapters/mysql2-adapter.ts`,
the `connected?` port) reads:

```ts
return this._client !== null && !this._permanentlyClosed && !this._isFakeConnection;
```

Rails' `Mysql2Adapter#connected?`
(`vendor/rails/activerecord/lib/active_record/connection_adapters/mysql2_adapter.rb:102`) is
just `!(@raw_connection.nil? || @raw_connection.closed?)` — handle presence and
the driver's own closed flag. The two extra terms are trails inventions:

- `_permanentlyClosed` — a trails-only latch set by `close()`, with no Rails
  ivar behind it. Rails models this by nulling `@raw_connection`.
- `_isFakeConnection` — the `fake_connection` constructor path keeps a handle
  slot populated in trails while Rails leaves `@raw_connection` nil until
  `verify!` promotes `@unconfigured_connection`.

Surfaced while collapsing `activeAsync` into `active` (PR #5967, story
`converge-adapter-active-predicate-to-async`): that PR removed the third
invented term (`_activeState`) and left these two, which predate it and were
out of its scope.

## Acceptance criteria

- Either the two terms are eliminated by making `_client` genuinely null in the
  permanently-closed and fake-connection states (Rails' modelling), or each
  surviving term carries a justification at the call site naming the trails
  mechanism that forces it.
- `Mysql2Adapter#isConnected` reads as a faithful port of `connected?`.
- The `active ⟹ isConnected` invariant and the fake-connection promotion path in
  `AbstractAdapter#verifyBang` keep working.

Related: [[rename-is-connected-q-onto-the-rails-connected-name]] renames the
method; this story converges its BODY.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
