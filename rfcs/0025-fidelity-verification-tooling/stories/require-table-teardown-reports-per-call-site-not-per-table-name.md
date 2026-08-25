---
title: "require-table-teardown dedupes per table name, making disables order-dependent"
status: draft
updated: 2026-08-25
rfc: "0025-fidelity-verification-tooling"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 90
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`blazetrails/require-table-teardown` reports at most one violation per table
NAME per file, anchored to the FIRST `createTable(...)` call for that name. Any
`// eslint-disable-next-line blazetrails/require-table-teardown` sitting on a
LATER call for the same name therefore becomes an "unused eslint-disable
directive" warning as soon as an earlier untorn-down creation of that name is
added — and conversely, deleting the earlier call re-reds the later one.

Surfaced in PR #7017
(`foreign-key-change-column-tables-bypass-migration-proper-table-name`). That PR
added `CreateRocketsMigration` near the top of
`packages/activerecord/src/migration/foreign-key.test.ts`, which creates
`rockets` / `astronauts`. The pre-existing disables in
`ForeignKeyInCreateTest > does not create foreign keys when bypassed by config`
(same file, `:memory:` connection, ~line 692/696) immediately turned into two
unused-directive warnings, with no change to that test at all. The PR kept both
sets: removing the older ones would re-red them under any future reordering.

`pnpm lint` runs without `--max-warnings`, so this is warnings-only today — but
the diagnostics are order-dependent, which makes them untrustworthy and makes
the disables impossible to place correctly.

## Converged shape

The rule reports per `createTable` CALL SITE, not per table name: each
untorn-down creation gets its own violation, and each `eslint-disable-next-line`
suppresses exactly the call it precedes. Whatever per-name dedupe exists today
for report volume must not make one call site's diagnostic depend on another's
presence.

## Acceptance criteria

- Two `createTable("x", ...)` calls in one file, both without teardown, produce
  two violations, and a disable on either suppresses only that one.
- `packages/activerecord/src/migration/foreign-key.test.ts` lints with zero
  unused-directive warnings and zero `require-table-teardown` errors.
- A rule test covers the repeated-name case in both orders.
