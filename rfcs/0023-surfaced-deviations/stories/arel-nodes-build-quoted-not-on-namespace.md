---
title: "Arel::Nodes.build_quoted is public in Rails but absent from trails' Nodes namespace"
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

Surfaced while converging arel's node tests in PR #7057 (RFC 0122).

Rails' `Arel::Nodes.build_quoted` is public module-level API —
`vendor/rails/activerecord/lib/arel/nodes/casted.rb:48`:

```ruby
def self.build_quoted(other, attribute = nil)
```

Rails' own tests call it as public surface, e.g.
`vendor/rails/activerecord/test/cases/arel/nodes/grouping_test.rb:9`:

```ruby
grouping = Grouping.new(Nodes.build_quoted("foo"))
```

trails HAS the function — `packages/arel/src/nodes/casted.ts:28`,
`export function buildQuoted(other: unknown, attribute?: unknown): Node` — but
`packages/arel/src/nodes/index.ts:6` re-exports only `{ Quoted, Casted }` from
that module, so `Nodes.buildQuoted` does not exist on the namespace a consumer
(or a mirrored test) sees. The ported grouping test had to reach past the
namespace with a direct `import { buildQuoted } from "./casted.js"`, which is
not the shape Rails' test has.

## Converged shape

Re-export `buildQuoted` from `packages/arel/src/nodes/index.ts` alongside
`Quoted` / `Casted`, so `Nodes.buildQuoted(...)` resolves the way
`Nodes.build_quoted` does in Ruby, and restore
`packages/arel/src/nodes/grouping.test.ts`'s call to go through the namespace.

Check the same treatment for any other `casted.ts` member Rails exposes at
module level before adding it — this is a missing-export fix, not a licence to
widen the namespace.

## Acceptance criteria

- [ ] `Nodes.buildQuoted` is reachable from the `Nodes` namespace, matching
      `Arel::Nodes.build_quoted`.
- [ ] `packages/arel/src/nodes/grouping.test.ts` calls it through the namespace,
      with no direct `./casted.js` import.
- [ ] `pnpm parity:api --package arel` delta non-negative and
      `pnpm parity:api:extra:gate` green (this ADDS a Rails-backed name, so
      arel's novel count must not move).
