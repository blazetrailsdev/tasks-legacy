---
title: "Preloader::ThroughAssociation by-owner readers should call the public recordsByOwner reader"
status: draft
updated: 2026-07-30
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
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

`Preloader::ThroughAssociation`'s `through_records_by_owner` /
`source_records_by_owner` reach their child loaders through the **public**
`records_by_owner` reader, which forces `load_records` before answering
(`vendor/rails/activerecord/lib/active_record/associations/preloader/association.rb:148-151`):

```ruby
def records_by_owner
  load_records unless defined?(@records_by_owner)
  @records_by_owner
end
```

trails cannot call that reader from these two methods: it is `async`, and they
are read synchronously from `_getMiddleRecords()` on the `runnableLoaders()` /
`futureClasses()` paths, which `Batch` drives synchronously
(`associations/preloader/batch.ts:44`). So after #5623 they read each child
loader's memoized map directly:

`packages/activerecord/src/associations/preloader/through-association.ts` —
`mergeRecordsByOwner(loaders)` reads `(loader as any)._recordsByOwner`.

Two compensations keep that sound, both trails-only inventions:

1. `recordsByOwner()` awaits every child's public reader before the synchronous
   merges run, restoring Rails' forcing on the one path that can await.
2. `mergeRecordsByOwner` returns `undefined` — never a partial merge — when any
   child has yet to load, so no caller can memoize an incomplete result.

Neither exists in Rails, and `_recordsByOwner` had to be widened from `private`
to `protected` on the base `Association` to make the peek possible.

## Acceptance criteria

- `through_records_by_owner` / `source_records_by_owner` obtain child maps
  through the public `recordsByOwner()` reader, so the forcing is intrinsic
  rather than hoisted into a separate step in `recordsByOwner()`.
- The `undefined`-on-incomplete guard in `mergeRecordsByOwner` and the eager
  forcing block in `recordsByOwner()` are both removed once the reader is the
  only path.
- `_recordsByOwner` returns to `private` on `Preloader::Association`.
- Requires making `middleRecords` / `runnableLoaders` / `futureClasses` async,
  or restructuring `Batch` so the middle-record derivation happens on an
  awaitable path — size the blast radius before starting.
- No regression in `associations.test.ts` (PreloaderTest),
  `associations/nested-through-associations.test.ts`,
  `associations/has-many-through-associations.test.ts`, and
  `associations/preloader/through-association-records-by-owner.trails.test.ts`
  (the direct-call regression test that pins the forcing).
