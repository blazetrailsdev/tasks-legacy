---
title: "Widen rails-file-structure-method-order to the connection-adapter tree"
status: ready
updated: 2026-08-25
rfc: "0119-connection-adapter-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 150
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`blazetrails/rails-file-structure-method-order` enforces Rails source order for
class members and top-level functions, and is autofixable. It is registered for
two packages only (`eslint.config.mjs:449`):

```js
files: ["packages/arel/src/**/*.ts", "packages/activemodel/src/**/*.ts"],
```

The whole `connection_adapters/` tree is therefore outside it, even though the
manifest already covers it — `eslint/rails-file-structure-method-order.json`
holds **84 adapter entries**, so the order data exists and is simply never
consulted.

This is the second half of gating the adapter tree. The first half (extra
surface) landed as #6997, which added `activerecord` to the
`parity:api:extra` ratchet.

### Measured 2026-08-24 against `main` (152b2ebe9)

Temporarily widening the glob to
`packages/activerecord/src/connection-adapters/**/*.ts` and
`packages/activerecord/src/adapters/**/*.ts`:

- **57 violations across 57 files.**
- `eslint --fix` resolves all 57 cleanly.
- The resulting diff is **12,744 insertions / 12,744 deletions** — 25,488 LOC of
  pure member reordering.

Files affected include `abstract-adapter.ts`, `abstract/connection-pool.ts`,
`abstract/schema-statements.ts`, `abstract/schema-definitions.ts`,
`abstract/transaction.ts`, `mysql2-adapter.ts`, `column.ts`, `deduplicable.ts`
and ~17 more.

### Why this is not one PR

25,488 LOC is far past the PR ceiling, and more importantly the reorder touches
nearly every file that RFC 0119's ~106 open stories, RFC 0106 and RFC 0073 are
all actively editing. Landing it as one change would conflict with essentially
every agent working in the tree.

Note that a pure reorder produces no semantic diff, so it is cheap to _redo_ but
expensive to _hold_ — it must land in a quiet window per slice, not be carried
on a long-lived branch.

## Acceptance criteria

- The lint glob covers `packages/activerecord/src/connection-adapters/**/*.ts`
  and `packages/activerecord/src/adapters/**/*.ts`, and `pnpm lint` is clean.
- Landed as a **sequence of slices**, each its own PR from `main`, each scoped to
  one subdirectory and taken when that area has no open PRs touching it.
  Suggested order, smallest blast radius first:
  1. `connection-adapters/sqlite3/**` and `sqlite3-adapter.ts`
  2. `connection-adapters/mysql/**`, `mysql2/**`, `mysql2-adapter.ts`,
     `abstract-mysql-adapter.ts`
  3. `connection-adapters/postgresql/**`, `postgresql-adapter.ts`
  4. `connection-adapters/abstract/**`
  5. the top-level files, and the glob widening itself as the final commit
- Each slice is `eslint --fix` output only — **no hand edits**, so the diff is
  verifiably a permutation. Reviewers should be able to confirm no line content
  changed.
- The glob widening lands **last**, so no intermediate slice leaves CI red.
- Before each slice: `gh pr list --search "<subdir>"` to confirm the area is
  quiet.

## Notes

File this as blocked-by-coordination rather than blocked-by-code: nothing
technical prevents it, but it needs a quiet window per slice.
