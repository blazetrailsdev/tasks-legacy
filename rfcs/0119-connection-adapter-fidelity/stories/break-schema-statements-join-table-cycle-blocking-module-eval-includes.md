---
title: "Break the schema-statements -> join-table -> model-schema cycle so AbstractAdapter's includes can return to the class body"
status: blocked
updated: 2026-08-25
rfc: "0119-connection-adapter-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 150
priority: null
pr: null
claim: "2026-08-25T17:17:28Z"
assignee: "converge-token-for-class-attribute-stores"
blocked-by: 'Measured: cutting migration/join-table.ts -> model-schema.ts with a zero-import slot DOES break the cycle (scripts/test-deps/ green with the mixin block back at module scope), but it reds the whole AR suite at collection time — ''Class extends value undefined'' at associations/collection-proxy.ts. The edge is also the accidental load-order guarantee the package relies on: the vitest setup preload entered associations.ts early through join-table -> model-schema -> associations, so associations.ts was always fully evaluated before relation.ts. Cut it and relation.ts -> insert-all.ts -> model-schema.ts -> associations.ts -> (associations.ts:6, a bare side-effect ''import "./associations/collection-proxy.js"'') -> relation.ts crashes with Relation still in TDZ. Approach 2 does not exist: every path from schema-statements.ts to abstract-adapter.ts runs through model-schema.ts, and model-schema.ts reaches abstract-adapter.ts by several independent legs; cutting model-schema -> connection-handling with a slot was tried and measured, and adapter-graph-import-tdz.test.ts still fails via model-schema -> associations -> persistence -> connection-handling. BLOCKER: associations.ts:6 must stop eagerly force-loading collection-proxy.ts (its ctor is already published through associations/collection-proxy-slot.ts) so relation.ts/associations.ts stop being load-order dependent. That is separate work on the relation<->associations cycle, not this story''s ~150 LOC.'
closed-reason: null
---

## Context

`abstract-adapter-mixin-wiring-restore-module-eval` is blocked (PR #7042) on a
module-eval cycle, and this story is the blocker itself.

Rails applies every AbstractAdapter mixin as a plain `include` in the class body
(`vendor/rails/activerecord/lib/active_record/connection_adapters/abstract_adapter.rb:50-56`):

```ruby
include Quoting, Savepoints, DatabaseLimits
include DatabaseStatements
include SchemaStatements
include QueryCache
```

trails defers all of it into `ensureAbstractAdapterMixinsApplied()`, run lazily
from the first `AbstractAdapter` construction
(`packages/activerecord/src/connection-adapters/abstract-adapter.ts`, the block
below the class). That deferral is a trails-only mechanism and it is what
`0025-fidelity-verification-tooling/extractor-detect-non-top-level-includes`
exists to see past.

PR #7042 measured what actually blocks the restore. Moving the block to module
scope and entering the graph at `SchemaStatements` throws
`TypeError: Cannot convert undefined or null to object` from
`packages/activesupport/src/include.ts` — `include(AbstractAdapter,
SchemaStatements)` reads `SchemaStatements` while it is still uninitialised, on:

```text
connection-adapters/abstract/schema-statements.ts
  -> migration/join-table.ts          (value import, joinTableName)
  -> model-schema.ts                  (value import, deriveJoinTableName)
  -> connection-handling.ts
  -> connection-adapters.ts
  -> connection-adapters/abstract-adapter.ts
```

PR #5775 removed the `... -> base.ts -> abstract-adapter.ts` leg that
`abstract-adapter.ts`'s comment used to cite (guarded by
`scripts/test-deps/base-import-cycle.test.ts`), but this leg survives it. The
comment in `abstract-adapter.ts` now names this cycle.

Note the first hop is Rails' own structure, not a trails invention:
`Migration::JoinTable#join_table_name` delegates to
`ModelSchema.derive_join_table_name`
(`vendor/rails/activerecord/lib/active_record/migration/join_table.rb:11-13`).
Ruby resolves that constant by autoload when the method runs, which is exactly
the call-time resolution ESM has no equivalent for.

## Converged shape

Break the cycle so `include(AbstractAdapter, …)` can run at module scope and the
class body matches abstract_adapter.rb:50-56.

Candidate approaches, in preference order:

1. Make the `migration/join-table.ts -> model-schema.ts` edge non-eager. This is
   the hop Ruby resolves at call time, so it is the honest place to defer — see
   CLAUDE.md, "Call-time constant resolution (Ruby autoload → the zero-import
   slot)". A slot module is the settled shape; confirm it actually closes the
   cycle before reaching for one.
2. If some other hop turns out to be a trails-only edge, cut that instead.

Do NOT re-document the deferral again — #7042 already did that, and the
deviation register is a burndown ledger, not permission.

## Acceptance criteria

- [ ] The mixin block runs at module-evaluation time;
      `ensureAbstractAdapterMixinsApplied` and its call site in the ctor are
      deleted.
- [ ] `pnpm vitest run scripts/test-deps/` passes — in particular
      `adapter-graph-import-tdz.test.ts` (enters at `SchemaStatements`),
      `base-import-cycle.test.ts`, and
      `pg-schema-definitions-import-tdz.test.ts`.
- [ ] Verified with a plain-node import of the BUILT `dist/**.js` modules as
      entry modules, not only under vitest — a vitest run enters the funnel
      module first and masks the TDZ.
- [ ] Unblock and close `abstract-adapter-mixin-wiring-restore-module-eval`.
- [ ] Check whether
      `0025-fidelity-verification-tooling/extractor-detect-non-top-level-includes`
      is still needed for this file.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.
