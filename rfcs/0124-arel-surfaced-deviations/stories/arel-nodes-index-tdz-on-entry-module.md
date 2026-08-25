---
title: "Two arel modules throw a TDZ error when entered as ESM entry modules"
status: ready
updated: 2026-08-25
rfc: "0124-arel-surfaced-deviations"
cluster: null
packages:
  - "arel"
deps: []
deps-rfc: []
est-loc: 120
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

# Two arel modules throw a TDZ error when entered as ESM entry modules

## Context

Found while verifying (per CLAUDE.md's "Call-time constant resolution" section)
that PR #6361's proposed `select-manager.ts → index.ts` import did not close a
new cycle. Importing the BUILT `dist/**.js` modules as entry modules under plain
node — the check CLAUDE.md mandates, because a vitest run enters the funnel
module first and masks it — shows two that already fail on `main`:

````text
node -e "import('packages/arel/dist/nodes/index.js')"
  -> Cannot access 'TableAlias' before initialization
node -e "import('packages/arel/dist/visitors/postgresql.js')"
  -> Cannot access 'TableAlias' before initialization
```text

`table.js`, `select-manager.js`, `index.js` and `nodes/unary.js` all load
cleanly, so the funnel entry points hide it. `visitors/postgresql.js` fails only
because it does `import * as Nodes from "../nodes/index.js"` — the underlying
defect is the cycle inside `nodes/index.ts` around `TableAlias`. This is
pre-existing and was NOT introduced by #6361 (verified by rebuilding with that
PR's arel changes reverted).

It matters because the whole point of the zero-import-slot doctrine is that
entering the graph at any module is safe; today a consumer deep-importing
`@blazetrails/arel/nodes` gets a hard `ReferenceError`.

## Converged shape

Break the `TableAlias` cycle in `packages/arel/src/nodes/index.ts` the way the
repo already breaks the other two (`encryption/configurable-slot.ts`,
`associations/collection-proxy-slot.ts`): a zero-import slot module, or a
reordering that removes the `class X extends Y` edge from the cycle.

## Acceptance criteria

1. `node --input-type=module -e "import('<dist>/nodes/index.js')"` and the same
   for `visitors/postgresql.js` both resolve, against the BUILT dist (a green
   vitest run does not count as evidence here).
2. Every other arel dist module still loads as an entry module — add the sweep
   as a check so this cannot regress silently.
3. No new runtime imports added to any zero-import slot.
````
