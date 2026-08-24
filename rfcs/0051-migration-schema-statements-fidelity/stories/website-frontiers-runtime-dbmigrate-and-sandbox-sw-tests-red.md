---
title: "website-frontiers-runtime-dbmigrate-and-sandbox-sw-tests-red"
status: ready
updated: 2026-08-24
rfc: "0051-migration-schema-statements-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: 61
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Two website frontiers tests fail on origin/main (verified at d2f3a1edc by
running main's own copy of the file), independent of the db/migrate rename in
the `website-frontiers-sandbox-still-scaffolds-db-migrations` story:

1. `packages/website/src/lib/frontiers/runtime.test.ts:131` —
   "exec: db:migrate (not yet supported) > errors explicitly when code
   execution is not available" expects the output to match
   `/not.*supported|sandboxed/i` (the message thrown by `executeCode` in
   `runtime.ts:56-60`), but `trail-cli.ts` `db:migrate` (`:215`) fails earlier
   inside `withMigrator` with `Error: No database connection defined.`, so the
   sandbox never reaches the eval-context error.
2. `packages/website/src/lib/frontiers/sandbox-sw.test.ts` — "sandbox-sw
   message handling > CLI execution > trail-cli accepts generate model command"
   times out at 5000ms.

## Acceptance criteria

- [ ] `db:migrate` in the frontiers sandbox surfaces the "requires a sandboxed
      eval context" error rather than "No database connection defined."
- [ ] The sandbox-sw "trail-cli accepts generate model command" test passes
      without a raised timeout, or its hang is root-caused and fixed.
- [ ] Test names unchanged.
