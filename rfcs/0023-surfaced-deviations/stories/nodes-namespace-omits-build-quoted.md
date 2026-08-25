---
title: "Nodes namespace omits build_quoted, forcing callers past it into casted.js"
status: draft
updated: 2026-08-25
rfc: "0023-surfaced-deviations"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 40
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Ruby reaches the quoted-node factory as `Arel::Nodes.build_quoted(...)` — it is
a module function on `Arel::Nodes` itself
(`vendor/rails/activerecord/lib/arel/nodes/casted.rb:48`, inside
`module Arel; module Nodes`), and Rails' own tests call it that way all over
`to_sql_test.rb` (`:213`, `:266`, `:283`, `:341`, …).

trails defines it faithfully at `packages/arel/src/nodes/casted.ts:28`, but
`packages/arel/src/nodes/index.ts:6` re-exports only `Quoted` and `Casted` from
that file, so the `Nodes` namespace users see — `export * as Nodes from
"./nodes/index.js"` (`packages/arel/src/index.ts:2`) — has no `buildQuoted`.
Every caller has to reach past the namespace into the module
(`import { buildQuoted } from "../nodes/casted.js"`), which is what
`nodes/casted.test.ts:4` and the converged `visitors/to-sql.test.ts` both do.

`parity:api` is satisfied (the member exists in the file matching `casted.rb`),
so nothing flags this today — it is a namespace-composition gap, not a missing
port.

## Converged shape

Add `buildQuoted` to the `nodes/index.ts` re-export beside `Quoted` / `Casted`,
so `Nodes.buildQuoted(...)` reads the way `Arel::Nodes.build_quoted(...)` does,
and switch the direct `../nodes/casted.js` imports in arel's own tests over to
it. Check `parity:api:extra:gate` before/after — arel is gated, and this adds a
name to the `Nodes` namespace surface (it should score as the existing Ruby
member, not as novel; if it scores novel, that is the finding).

## Acceptance criteria

- `Nodes.buildQuoted` resolves, matching `Arel::Nodes.build_quoted`
  (`arel/lib/arel/nodes/casted.rb:48`).
- No arel test imports `buildQuoted` from `../nodes/casted.js` any more.
- `pnpm parity:api:extra:gate` stays green (arel novel does not increase).
