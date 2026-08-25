---
title: "callbacks-update-callbacks-reads-its-own-descendants"
status: draft
updated: 2026-08-23
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activesupport"
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`callbacks.rb:687` is `self.descendants.prepend(self)` inside
`__update_callbacks`, which walks its own descendant list. trails'
`__updateCallbacks` (`packages/activesupport/src/callbacks.ts`) takes the target
list as a `targets` parameter because every caller already holds it — a TS class
has no `DescendantsTracker` registration at this layer — so neither
`descendants` nor `prepend` is called.

Both calls now sit at the call site as `@missingRailsCall` receipts opened
`CONVERGEABLE (story callbacks-update-callbacks-reads-its-own-descendants)`,
migrated out of `call-mismatches-exclude/activesupport/callbacks.json` by
RFC 0106 wave 5g.

## Acceptance criteria

- [ ] `__updateCallbacks` reads its descendant list itself — a `descendants`
      reader on the callbacks host — and prepends `self`, matching
      `callbacks.rb:685-690`.
- [ ] The `targets` parameter is gone and every caller updated.
- [ ] Both `@missingRailsCall` receipts deleted, not reworded.
- [ ] `pnpm parity:api:calls` and `pnpm parity:api:calls:args` green.
