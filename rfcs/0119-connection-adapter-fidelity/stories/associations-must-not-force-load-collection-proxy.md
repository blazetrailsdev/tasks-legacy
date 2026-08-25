---
title: "associations.ts must stop force-loading collection-proxy.ts at module scope (~150 LOC)"
status: claimed
updated: 2026-08-25
rfc: "0119-connection-adapter-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 150
priority: null
pr: null
claim: "2026-08-25T18:50:35Z"
assignee: "pg-column-serial-identity-fields-are-public-where-rails-has-ivars"
blocked-by: null
closed-reason: null
---

# associations.ts must stop force-loading collection-proxy.ts at module scope

## Context

Surfaced by PR #7056 while attempting
`break-schema-statements-join-table-cycle-blocking-module-eval-includes`, which
is now blocked on this.

`packages/activerecord/src/associations.ts:6` is a bare side-effect import:

```ts
import { _CollectionProxyCtor } from "./associations/collection-proxy-slot.js";
import "./associations/collection-proxy.js";
```

The slot on the line above it (`associations/collection-proxy-slot.ts`) is the
repo's settled zero-import shape for exactly this — CLAUDE.md, "Call-time
constant resolution (Ruby autoload → the zero-import slot)" names it as one of
the two sanctioned instances. But line 6 then force-loads the module the slot
exists to defer, so `associations.ts` cannot be evaluated unless `relation.ts`
is already fully evaluated: `collection-proxy.ts`'s class `extends Relation`.

That makes the whole package entry-order dependent, and today it only works by
luck. PR #7056 measured it: replace `migration/join-table.ts -> model-schema.ts`
with a zero-import slot and `scripts/test-deps/` goes green with
AbstractAdapter's `include`s restored to module scope (abstract_adapter.rb:50-56),
but every ActiveRecord suite reds at COLLECTION time with

```text
TypeError: Class extends value undefined is not a constructor or null
  packages/activerecord/src/associations/collection-proxy.ts
  packages/activerecord/src/associations.ts:6
```

because the vitest setup preload used to enter `associations.ts` early through
join-table -> model-schema -> associations, so `associations.ts` was always
fully evaluated before `relation.ts`. With that edge cut, the entry becomes

```text
relation.ts -> insert-all.ts -> model-schema.ts -> associations.ts
  -> (associations.ts:6) collection-proxy.ts -> relation.ts   [Relation in TDZ]
```

Ruby has no equivalent problem: `CollectionProxy` is named inside method bodies
and Zeitwerk autoloads it when they run
(`activerecord/lib/active_record/associations/collection_association.rb`
`reader` -> `CollectionProxy.create`), which is precisely what the slot models.

## Converged shape

Delete the `import "./associations/collection-proxy.js"` side-effect line from
`associations.ts` and let `collection-proxy.ts` be reached the way the slot
intends — the module that defines `CollectionProxy` registers itself through
`_setCollectionProxyCtor` at the bottom of its own body, and whoever needs the
ctor reads the slot binding at call time. Whatever currently guarantees
`collection-proxy.ts` is evaluated at all must move to a module that does not
sit inside the relation<->associations cycle (entering at `relation.ts` must not
require `associations.ts`, and vice versa).

Verify BOTH, because either alone lies:

- `pnpm vitest run scripts/test-deps/` (in particular
  `adapter-graph-import-tdz.test.ts`, `base-import-cycle.test.ts`), and
- a real AR suite file, e.g.
  `pnpm vitest run packages/activerecord/src/migration/columns.test.ts` — a cut
  that greens test-deps can red every AR suite at collection time.
- a plain-node import of the BUILT `dist/**.js` modules as entry modules; a
  vitest run enters the funnel module first and masks the TDZ.

## Acceptance criteria

- [ ] `associations.ts` carries no bare side-effect import of
      `associations/collection-proxy.js`.
- [ ] Entering the graph at `relation.ts` and at `associations.ts` both evaluate
      cleanly, verified with plain-node `dist` entry-module imports.
- [ ] `pnpm vitest run scripts/test-deps/` green; SQLite, PostgreSQL and
      MySQL/MariaDB lanes green.
- [ ] Unblock `break-schema-statements-join-table-cycle-blocking-module-eval-includes`
      (and through it `abstract-adapter-mixin-wiring-restore-module-eval`).
