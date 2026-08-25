---
title: "Converge the owner-level _strictLoadingBypassCount onto Rails' association-level @skip_strict_loading"
status: draft
updated: 2026-08-13
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

Rails suppresses a strict-loading raise with an association-level ivar:
`Association#skip_strict_loading` sets `@skip_strict_loading`, and
`violates_strict_loading?` opens with `return if @skip_strict_loading`
(vendor/rails/activerecord/lib/active_record/associations/association.rb:276-285).
The only callers are `CollectionAssociation#concat` (collection_association.rb:130)
and `#replace` (collection_association.rb:244).

trails carries a SECOND, owner-level mechanism with no Rails counterpart: the
`_strictLoadingBypassCount` on `Base` (base.ts:2979), bumped by the explicit
loaders `loadBelongsTo` / `loadHasOne` (associations.ts:1598/1621,
associations/instance-methods.ts:125-129) and by
`CollectionProxy#_withoutStrictLoading` (associations/collection-proxy.ts:1161).

PR #6472 wired `Association#isViolatesStrictLoading` into the find_target call
sites, and had to make its first guard read BOTH flags
(associations/association.ts) to keep the explicit loaders working — a
deliberate, cited deviation, but a deviation.

## Acceptance criteria

1. The explicit loaders suppress the raise the Rails way: they obtain the
   association instance and run their load inside `skipStrictLoading(...)`
   (`Association#skip_strict_loading`), rather than bumping a counter on the
   owner record.
2. `Base._strictLoadingBypassCount` is deleted, along with the second arm of
   the guard in `Association#isViolatesStrictLoading` — that body ends up a
   line-for-line port of association.rb:284-292.
3. `packages/activerecord/src/strict-loading-sync-reader.test.ts` and
   `strict-loading.test.ts` stay green (the explicit-load "per-call mute" tests
   are the behavioral contract).
