---
title: "seedAssociationCache is a trails-only test helper with no Rails counterpart"
status: claimed
updated: 2026-08-24
rfc: "0112-one-rails-thing-n-trails-things"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 200
priority: null
pr: null
claim: "2026-08-24T12:03:42Z"
assignee: "extra-surface-allow-reopened-module-method-files"
blocked-by: null
closed-reason: null
---

## Context

Surfaced landing PR #6975. `packages/activerecord/src/support/seed-association-cache.ts`
exists only to poke a loaded target into a record's association cache from a
test. After #6975 its whole body is

```ts
const assoc = record.association(name);
assoc._setTargetFromLoader(target);
assoc._explicitTarget = true;
```

Rails has no such helper. Its tests reach the same state the production way —
`Topic.includes(:replies).first` / `preload`, or building through the
association — and read `topic.association(:replies).loaded?` directly. The
helper survives in five test files
(`validations/absence-validation.test.ts`, `validations/association-validation.test.ts`,
`validations/i18n-validation.test.ts`, `validations/presence-validation.test.ts`,
`strict-loading-sync-reader.test.ts`), and it is the last consumer of the
trails-only `_explicitTarget` flag outside `associations.ts` (see
`cache-singular-target-inverse-write-bypasses-inversed-from`).

## Converged shape

Each call site reaches its loaded state the way its Rails counterpart test does
(fixtures + `includes`/`preload`, or an association build), and
`support/seed-association-cache.ts` is deleted.

## Acceptance criteria

- [ ] `seedAssociationCache` is gone, with no replacement helper.
- [ ] Every affected test still asserts what its Rails counterpart asserts, with
      unchanged test names.
