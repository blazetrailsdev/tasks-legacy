---
title: "counter-cache: counterCachedAssociationNames falls back to scanning _associations"
status: done
updated: 2026-08-22
rfc: "0112-one-rails-thing-n-trails-things"
cluster: split-stores
deps: []
deps-rfc: []
est-loc: 70
pr: 6777
claim: "2026-08-20T17:22:15Z"
assignee: "converge-event-children-invention"
blocked-by: null
closed-reason: null
---

## Context

`counterCachedAssociationNames` in
`packages/activerecord/src/counter-cache.ts` falls back to scanning
`ctor._associations` for `belongsTo` entries with `counterCache` whenever the
`_counterCachedAssociationNames` registry is empty. Rails has no fallback: the
list is purely the `class_attribute` unioned by
`Builder::BelongsTo.add_counter_cache_callbacks`
(`vendor/rails/activerecord/lib/active_record/associations/builder/belongs_to.rb:27-41`),
read directly in `_create_record` / `destroy_row`
(`counter_cache.rb:200-224`).

The fallback also needs a `new Set(...)` dedupe (added by #5373) that Rails does
not, because `_associations` keeps both an inherited and an overriding
`belongs_to` of the same name where Rails' reflections hash keys by name.

## Acceptance criteria

- Every `belongs_to ..., counter_cache:` declaration registers through
  `registerCounterCachedAssociation` at declaration time (the builder path), so
  the registry is authoritative.
- The `_associations` scan and its dedupe are deleted from
  `counterCachedAssociationNames`.
- `counter-cache.test.ts` and `associations/belongs-to-associations.test.ts` stay
  green.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
