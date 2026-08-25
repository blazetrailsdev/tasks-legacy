---
title: "Restore AbstractAdapter mixin wiring to module-evaluation time now that base.ts is out of the cycle"
status: claimed
updated: 2026-08-25
rfc: "0119-connection-adapter-fidelity"
cluster: null
deps: []
deps-rfc: []
est-loc: 120
priority: null
pr: null
claim: "2026-08-25T15:38:33Z"
assignee: "pg-table-definition-takes-unlogged-as-an-option-rails-reads-the-adapter"
blocked-by: null
closed-reason: null
---

## Context

`packages/activerecord/src/connection-adapters/abstract-adapter.ts:2725-2745`
defers its entire mixin block (`include(AbstractAdapter, DatabaseStatements)`,
`include(AbstractAdapter, SchemaStatements)`, `include(AbstractAdapter,
QuotingMixin)`, `include(AbstractAdapter, QueryCacheMixin)`, the `setCallback`
registrations, the `Savepoints`/`DatabaseLimits` includes and the cached
`selectAll` rewrap) into `ensureAbstractAdapterMixinsApplied()`, run lazily from
the first `AbstractAdapter` construction. Its own comment states why:

> Applying it eagerly at module-evaluation time re-references the mixin classes
> (SchemaStatements, DatabaseStatements, …) which, on a circular import edge
> (schema-types -> SchemaStatements -> schema-dumper -> base -> abstract-adapter),
> are still in their temporal dead zone.

Rails does all of these as plain `include`s in the class body
(`activerecord/lib/active_record/connection_adapters/abstract_adapter.rb:50-56`),
so the deferral is a trails-only mechanism — and
`0025-fidelity-verification-tooling/extractor-detect-non-top-level-includes`
exists precisely because it hides these includes from the extractor.

PR #5775 removed the `... -> base -> abstract-adapter` leg of that cycle:
`base.ts` is no longer a member of any import cycle, guarded by
`scripts/test-deps/base-import-cycle.test.ts`. The cited justification for the
deferral may therefore no longer hold. This story is to determine whether the
lazy block can go back to module-evaluation time (matching Rails' class body),
or — if a residual cycle not involving `base.ts` still makes it unsafe — to
re-document the deferral against the real remaining cycle rather than the stale
`-> base ->` one.

## Acceptance criteria

- Either the mixin block runs at module-evaluation time again (Rails' class-body
  `include` shape, `ensureAbstractAdapterMixinsApplied` and its call sites
  deleted), or the comment at abstract-adapter.ts:2725 cites the actual
  remaining cycle and the story is closed as blocked with that cycle named.
- `pnpm vitest run scripts/test-deps/adapter-graph-import-tdz.test.ts` and
  `scripts/test-deps/base-import-cycle.test.ts` pass, and the adapter graph
  still imports cleanly when entered through `SchemaStatements`.
- If the block moves back to module scope, check whether
  `0025-fidelity-verification-tooling/extractor-detect-non-top-level-includes`
  is still needed for this file.
