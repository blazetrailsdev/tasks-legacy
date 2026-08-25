---
title: "Converge performFind raise paths onto raiseRecordNotFoundExceptionBang"
status: ready
updated: 2026-07-27
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

Surfaced by PR 5055 (converge-record-not-found-conditions-onto-arel-where-sql).
Refines the done umbrella story
performfind-converge-raise-record-not-found-builder (PR 4665), which converged
the message shape but not the raise routing.
Rails' `find_one` / `find_some` / `find_some_ordered`
(`vendor/rails/activerecord/lib/active_record/relation/finder_methods.rb:~440-480`)
raise via `raise_record_not_found_exception!(ids, result_size, expected_size)`,
which builds the conditions string itself. trails' `performFind`
(`packages/activerecord/src/relation/finder-methods.ts:306-360`) instead
precomputes `conditions` up front (now via the same Rails expression, PR 5055) and routes raises through the trails-only `raiseNotFoundSingle` /
`raiseNotFoundAll` helpers (`finder-methods.ts:192-260`), shared with
`collection-association.ts:189` (ids_writer) and `collection-proxy.ts` (the
in-memory find path).

Known message divergence hiding behind the helpers: for composite PKs,
`raiseNotFoundAll` renders `String(tuples)` (`"1,2,3,4"`), while Rails'
`raise_record_not_found_exception!` does `ids.join(", ")` — Ruby's `Array#join`
recursively joins nested arrays with the separator (`"1, 2, 3, 4"`). The bang's
multi branch (`wrapped.join(", ")`) gives yet another shape for nested arrays
(`"1,2, 3,4"`).

## Acceptance criteria

- [ ] `performFind`'s raise paths call `raiseRecordNotFoundExceptionBang`
      (via `.call(this, ids, resultSize, expectedSize, key)`), deleting the
      precomputed `conditions` local.
- [ ] Composite-PK multi-id message ids render Ruby-`join`-faithfully
      (`"1, 2, 3, 4"` for `[[1,2],[3,4]]`), verified against Rails behavior.
- [ ] `raiseNotFoundSingle` / `raiseNotFoundAll` are deleted or reduced to the
      minimum the collection-association/proxy callers need, with call-site
      justification for whatever remains.
- [ ] RecordNotFound message assertions across activerecord still pass.
