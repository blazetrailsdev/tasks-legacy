---
title: "Port Ruby's Array#drop for the two reflection-chain call sites"
status: done
updated: 2026-08-24
rfc: "0106-wide-call-set-direct-burndown"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 60
priority: 3
pr: 6968
claim: "2026-08-24T02:45:38Z"
assignee: "sweep-trails-only-test-files-relation"
blocked-by: null
closed-reason: null
---

## Context

Two call-set deviations spell Ruby's `Enumerable#drop(1)` as a JS
`Array#slice(1)`, so the ported body makes no `drop` call and the call-set gate
(RFC 0047) has nothing to credit:

- `packages/activerecord/src/associations/through-association.ts` `targetScope`
  — Rails `reflection.chain.drop(1)`
  (`vendor/rails/activerecord/lib/active_record/associations/through_association.rb:36`).
  Now carries a `@missingRailsCall drop` receipt naming this story.
- `packages/activerecord/src/associations/association-scope.ts` `getChain` —
  Rails `chain.drop(1)`
  (`vendor/rails/activerecord/lib/active_record/associations/association_scope.rb`),
  still a `call-mismatches-exclude` row.

The repo already has the settled shape for a Ruby core collection method with
no JS spelling: `packages/activesupport/src/ruby-empty.ts` (`empty?`) and
`packages/activerecord/src/ruby-first.ts` (`first`), both written precisely so a
faithfully ported body emits a call the gate can see.

## Acceptance criteria

- A `drop` helper lands beside `ruby-empty.ts` / `ruby-first.ts`, porting only
  the Array arm the callers need, with the same `@internal` framing and the
  same "why this exists" JSDoc.
- Both call sites above call it, so `pnpm parity:api:calls` credits `drop`.
- The `@missingRailsCall drop` tag on `targetScope` and the
  `association-scope.ts` `get_chain drop` baseline row are both deleted (the
  baseline is only-shrink; delete the row by hand, then
  `pnpm parity:api:calls:tighten`).
