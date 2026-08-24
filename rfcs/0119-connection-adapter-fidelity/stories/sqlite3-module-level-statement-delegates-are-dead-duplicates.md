---
title: "sqlite3 module-level schema/database-statements delegates duplicate the adapter and never run"
status: draft
updated: 2026-08-02
rfc: "0119-connection-adapter-fidelity"
cluster: null
deps: []
deps-rfc: []
est-loc: 200
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`packages/activerecord/src/connection-adapters/sqlite3/schema-statements.ts`
says so in its own header comment: `addForeignKey`, `removeForeignKey`,
`checkConstraints`, `addCheckConstraint` and `removeCheckConstraint` "are
implemented on SQLite3Adapter directly (via alterTable rebuild). The functions
below delegate to the adapter." Nothing imports those module-level exports —
`AbstractSQLite3Adapter` defines its own methods and never routes through them.
The same is true of the transaction entries in
`connection-adapters/sqlite3/database-statements.ts`
(`beginDbTransaction` / `beginDeferredTransaction` / `beginIsolatedDbTransaction`
/ `internalBeginTransaction`): PR #5913 converged them onto a shared primitive
purely to clear wide-call baseline entries, and the reviewer on that PR
confirmed they are "dead code at runtime (adapter class doesn't import these)".

So the same logic is maintained twice, and only one copy runs. That is an active
hazard, not just clutter: #5913 shipped a real bug where the adapter's
`removeCheckConstraint` dropped its third `options` parameter, and the dead
delegate carried the identical two-parameter signature — it had to be fixed
separately in the same PR, and would silently have reintroduced the defect had
anything ever wired it up.

## Acceptance criteria

- Decide the endgame for each dead export: either wire the adapter to actually
  call it (matching Rails' module-mixin layout per CLAUDE.md's `this`-typed
  mixin convention), or delete it and record why the Rails-layout slot is
  intentionally empty.
- Do NOT simply delete to make the file smaller — the exports exist so
  `parity:api` finds the method in the file matching Rails' layout. Removing
  them without a replacement will regress the parity:api file-structure
  numbers; check `pnpm parity:api:extra` and `pnpm parity:api:calls` both ways.
- Whichever direction, the logic must exist exactly once afterwards.
- Scope check before starting: this may exceed the LOC ceiling across both
  files. Split by file if so.
