---
title: "BatchEnumerator should not carry a generator field"
status: draft
updated: 2026-08-15
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 200
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

# `BatchEnumerator` should not carry a generator at all

## Context

`vendor/rails/activerecord/lib/active_record/relation/batches/batch_enumerator.rb:8-16`
stores only the seven kwargs; `#each` (:104-108) and `#each_record` (:54-57)
re-derive their batches with `@relation.to_enum(:in_batches, ...)`, i.e. by
calling `in_batches` **with a block**. There is no generator on the object.

PR #6561 converged the constructor to those seven kwargs, but
`packages/activerecord/src/relation/batches/batch-enumerator.ts:36` still
carries an `@internal _generator` field that
`packages/activerecord/src/relation.ts` (`inBatches`) assigns after
construction, and `each`/`eachRecord` read it back off the enumerator that
`relation.inBatches(...)` returns. `findInBatches` (relation.ts) also reaches
for `enumerator._generator()` as its spelling of Rails' block arm.

`in_batches` now HAS the block arm (`batches.rb:258,267`), so the shape the
generator works around may no longer be needed: `each(fn)` can call
`relation.inBatches(opts, fn)` directly. What still has no Ruby counterpart is
the BLOCKLESS `each()` / `[Symbol.asyncIterator]` — Ruby returns a lazy
Enumerator there and JS has no pull-based equivalent for an async block arm
without a queue.

## Converged shape

`BatchEnumerator` holds the seven kwargs and nothing else. `each(fn)` /
`eachRecord(fn)` call `relation.inBatches(...)` with the block. The blockless
async-iteration path keeps whatever minimum surface it genuinely needs, tagged
`@noRailsEquivalent PERMANENT` at that one site — not a mutable field the
relation reaches into.

Fold in the remaining novel surface on the class while there:
`eachBatch` (no Ruby counterpart; `each` is the whole surface) and `toArray`
(Ruby gets `to_a` from `include Enumerable`).

## Acceptance criteria

- [ ] `_generator` is gone, or reduced to a single tagged site with no
      cross-module assignment from `relation.ts`.
- [ ] `findInBatches` goes through the block arm rather than a generator handle.
- [ ] `pnpm parity:api:extra --package activerecord` shows no more novel names
      on `relation/batches/batch-enumerator.ts` than today (1).
- [ ] `batches.test.ts` + `batches.trails.test.ts` green on all three lanes.
