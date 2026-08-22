---
title: "Preloader::Batch#loaders attr_reader is missing from the port"
status: draft
updated: 2026-08-22
rfc: "0023-surfaced-deviations"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 30
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`pnpm parity:api --package activerecord` reports 2 missing methods; one is
`ActiveRecord::Associations::Preloader::Batch#loaders`
(`vendor/rails/activerecord/lib/active_record/associations/preloader/batch.rb:37`),
the private `attr_reader :loaders` sitting under `private` next to
`group_and_load_similar` (`:39`).

`packages/activerecord/src/associations/preloader/batch.ts` has no counterpart —
it keeps `_preloaders` / `_availableRecords` and threads the loaders through as
locals.

## Converged shape

The `attr_reader`'s trails spelling: a `loaders` member on `Batch` at the Rails
position in the file (private half of the class), read where Rails reads it.
Check what `attr_reader :loaders` actually backs in Rails first — `@loaders` is
never assigned in `batch.rb`, so the converged shape may be the reader over a
field the port also has to introduce, or the Rails-faithful no-op reader.

## Acceptance criteria

- `preloader/batch.ts` mirrors `batch.rb`'s member list, including `loaders`.
- `pnpm parity:api --package activerecord` loses the `loaders` missing row.
- `pnpm vitest run packages/activerecord/src/associations/preloader` green.
