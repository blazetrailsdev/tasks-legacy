---
title: "in_batches' loaded? arm must drain a scheduled relation"
status: claimed
updated: 2026-08-23
rfc: "0107-relation-ts-decomposition"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 120
priority: null
pr: null
claim: "2026-08-23T15:42:31Z"
assignee: "excluding-must-drain-a-scheduled-relation"
blocked-by: null
closed-reason: null
---

# `in_batches`' loaded? arm must drain a scheduled relation

## Context

PR #6918 gave `load_async` Rails' `@loaded = true` alongside `@future_result`
(`vendor/rails/activerecord/lib/active_record/relation.rb:1149`), so a
scheduled relation is now `loaded?` with its rows still parked.

`Batches#in_batches` (`packages/activerecord/src/relation/batches.ts:193`)
picks between `batchOnLoadedRelation` and the querying arm SYNCHRONOUSLY, at
call time, and `batchOnLoadedRelation` (`batches.ts:432-441`) reads
`relation._records` synchronously. So #6918 had to spell the arm as
`this.loaded && !this.isScheduled`, which sends a scheduled relation down the
querying arm — Rails
(`vendor/rails/activerecord/lib/active_record/relation/batches.rb`, the
`if loaded?` arm of `in_batches`) drains the future and batches in memory.

This is NOT a language shortcoming: the branch does not have to be taken
synchronously. Every consumer goes through the async generator
(`batches.ts:208`, `:221`) or `BatchEnumerator`, both of which can `await`.

## Converged shape

Move the `loaded?` decision (and the `batchOnLoadedRelation` call) INSIDE the
async generator, awaiting `relation.records()` instead of reading `_records`,
so `loaded?` alone is the guard — matching Rails' `in_batches` — and a
`load_async` relation batches the drained rows rather than re-querying. Drop
the `!this.isScheduled` conjunct and its call-site comment.

## Acceptance criteria

- [ ] `in_batches` guards on `loaded?` alone, as Rails does.
- [ ] `batchOnLoadedRelation` reads the `records` seam, not `_records`.
- [ ] A `loadAsync()` relation put through `inBatches` / `findInBatches` /
      `findEach` issues NO query beyond the scheduled one — regression test
      failing on the baseline.
- [ ] `parity:api:calls` / `:args` / `:extra` clean.
