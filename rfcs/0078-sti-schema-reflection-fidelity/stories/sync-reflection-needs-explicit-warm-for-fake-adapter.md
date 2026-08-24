---
title: "Sync schema reflection silently no-ops for a cold fake-adapter model"
status: ready
updated: 2026-08-24
rfc: "0078-sti-schema-reflection-fidelity"
cluster: null
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

`packages/activerecord/src/test-helpers/models/contact.ts` ends with two
explicit `await Klass.loadSchema()` calls that Rails' `contact.rb` has no
analogue for. Rails reflects the fake adapter's columns lazily on the first
`columns` read; trails' synchronous reflection path
(`loadSchemaFromCacheSync`, model-schema.ts) reads only an ALREADY-WARM schema
cache and silently falls through otherwise, so a model whose columns come from
`merge_column` has an empty attribute set until something awaits the async
`loadSchemaFromAdapter`.

Removing either call was verified to break `json-serialization.test.ts` and
`serialization.test.ts` (15 failures), so the warm-up is load-bearing, not
decorative — the deviation is in the reflection path, not the model.

Any future model that gets its columns from a non-DB adapter hits the same
trap, and the warm-up is easy to forget because the failure surfaces as
missing attributes far from the cause.

## Acceptance criteria

- The synchronous reflection path either warms itself for an adapter whose
  `columns` is synchronous (the fake adapter's shape), or fails loudly instead
  of returning an empty attribute set.
- `contact.ts` drops the two `loadSchema()` calls and still passes
  `json-serialization.test.ts` and `serialization.test.ts`.
- No behaviour change for DB-backed models, whose cache is warm via RFC 0031.
