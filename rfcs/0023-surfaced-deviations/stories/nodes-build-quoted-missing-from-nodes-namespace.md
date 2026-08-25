---
title: "Nodes.buildQuoted is missing from the Nodes namespace re-export"
status: draft
updated: 2026-08-25
rfc: "0023-surfaced-deviations"
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

Rails exposes `build_quoted` as a member of the `Arel::Nodes` module —
`vendor/rails/activerecord/lib/arel/nodes.rb:9` requires `nodes/casted`, and
`vendor/rails/activerecord/lib/arel/nodes/casted.rb:47-58` defines
`Arel::Nodes.build_quoted` as a module function. Rails' own arel tests call it
that way, e.g.
`vendor/rails/activerecord/test/cases/arel/visitors/mysql_test.rb:31`
(`Nodes::Limit.new(Nodes.build_quoted("omg"))`) and
`sqlite_test.rb:50` (`Nodes.build_quoted(nil, @table[:active])`).

trails defines `buildQuoted` in `packages/arel/src/nodes/casted.ts` but does not
re-export it from `packages/arel/src/nodes/index.ts`, so there is no
`Nodes.buildQuoted`. Every call site reaches past the namespace with a direct
module import — `import { buildQuoted } from "../nodes/casted.js"` — including
the two visitor test files converged in #7016
(`packages/arel/src/visitors/mysql.test.ts`,
`packages/arel/src/visitors/sqlite.test.ts`), where the Ruby they mirror says
`Nodes.build_quoted`.

Adding the re-export was attempted in #7016 and reverted: it moved arel's
extra-surface measurement from novel 0 / total 63 to novel 1 / total 65 and
turned `pnpm parity:api:extra:gate` (RFC 0117) red. The gate is only-shrink, so
the name has to be recognised as having a Ruby counterpart rather than counted
as novel surface — `build_quoted` is a module function on `Arel::Nodes`, and the
extractor evidently does not pair it with a TS re-export of the same name.

## Converged shape

`Nodes.buildQuoted` resolves, mirroring `Arel::Nodes.build_quoted`
(`arel/nodes/casted.rb:47-58`), and call sites spell it `Nodes.buildQuoted`
rather than importing `buildQuoted` from `nodes/casted.js` directly. This
requires the RFC 0117 extra-surface extractor to match the re-export against the
Ruby module function; the fix belongs there, not in a widened mark.

## Acceptance criteria

- [ ] `packages/arel/src/nodes/index.ts` re-exports `buildQuoted`.
- [ ] `pnpm parity:api:extra:gate` is green with arel's mark unchanged — the
      name is matched to `arel/nodes/casted.rb`, not absorbed by raising the
      mark.
- [ ] The mirrored visitor tests spell it `Nodes.buildQuoted`, as their Ruby
      counterparts do.
