---
title: "relation.js TDZ-crashes as an entry module"
status: draft
updated: 2026-08-22
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 80
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

# `relation.js` TDZ-crashes as an entry module

## Context

Importing the built module directly throws before any trails code runs:

    node -e "import('./packages/activerecord/dist/relation.js')"
    # Cannot access 'Relation' before initialization

`packages/activerecord/dist/index.js` and `dist/base.js` both import cleanly, so
the graph only breaks when `relation.js` is the entry.

Found while verifying that PR #6838's new `reflection.ts` -> `model-schema.ts`
import (converging `derive_join_table` onto
`ModelSchema.derive_join_table_name`, `activerecord/lib/active_record/reflection.rb:879-881`)
did not introduce a cycle. It did not. **A/B-confirmed pre-existing on
`origin/main`**: reverting only that import and rebuilding still reproduces the
crash. Same family as `arel-nodes-index-tdz-on-entry-module` and
`builder-association-tdz-on-entry-module`.

Likely introduced with `Relation.create` / `relationClassFor`
(PR #6840, converging `build_scope` at `reflection.rb:336-338`), which added a
`class Sub extends Relation` edge reachable from `relation.ts` itself — the
shape CLAUDE.md's "Call-time constant resolution" section describes: entering
the graph at the superclass module evaluates a subclass with `Relation` still in
TDZ.

`scripts/test-deps/adapter-graph-import-tdz.test.ts` and
`base-import-cycle.test.ts` both pass, so the guards do not cover this entry
point — a vitest run enters the funnel module first and masks it, which is why
CLAUDE.md requires a plain-node import of the BUILT `dist/**.js` module.

## Converged shape

Find the `class Sub extends Relation` edge that closes the cycle and break it
the sanctioned way — the zero-import slot module (CLAUDE.md, "Call-time
constant resolution"): a file with no runtime imports exporting a mutable
binding plus a `_setX()` setter, which the defining module calls at the bottom
of its own body. `associations/collection-proxy-slot.ts` and
`encryption/configurable-slot.ts` are the two existing instances.

Do NOT defer the subclass edges instead (a slot per `extends` site): nothing
then loads the subclass modules at all, so their self-registration never runs.

## Acceptance criteria

- [ ] `node -e "import('./packages/activerecord/dist/relation.js')"` resolves.
- [ ] `dist/index.js` and `dist/base.js` still import cleanly as entry modules.
- [ ] A guard covers `relation.js` as an entry module, so this cannot regress
      silently — extend `scripts/test-deps/adapter-graph-import-tdz.test.ts`
      rather than adding a new harness.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.
