---
title: "Plumb associationForeignKey into the HABTM join model and join SQL"
status: ready
updated: 2026-07-27
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 120
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`associationForeignKey` is retained on the HABTM reflection options to mirror Rails'
`habtm_reflection` (which keeps the full options hash), but it is not plumbed into
the generated join model or the join SQL. `_build` and `_resolveHabtmJoin` in
`packages/activerecord/src/associations/builder/has-and-belongs-to-many.ts`
hard-code the target FK as `${singular(name)}_id`.

Rails derives it in `Builder::HasAndBelongsToMany#belongs_to_options`
(`vendor/rails/activerecord/lib/active_record/associations/builder/has_and_belongs_to_many.rb`):
explicit `:association_foreign_key` wins, else `class_name.foreign_key`, else the
default `belongs_to` derivation. trails has that precedence implemented in
`habtmTargetFk` (`packages/activerecord/src/associations.ts`) and the builder does
call it for the join-model side — but the divergence noted in the in-source comment
is that full plumbing into the generated join model and join SQL was deferred.

Surfaced while deleting the dead `loadHabtm` (PR #5362); the now-removed loader was
one of the call sites the comment named, so the comment was updated but the
underlying gap remains. Tracked here rather than left as prose in a code comment.

## Acceptance criteria

- Determine whether the deferral is still real after `loadHabtm`'s removal — if
  `habtmTargetFk` already feeds every surviving join-key path, close this as
  already-converged and delete the stale "follow-up" comment in
  `builder/has-and-belongs-to-many.ts`.
- If real, plumb `associationForeignKey` through the generated join model and join
  SQL so a non-default target FK is honored end to end.
- Cover with a canonical-schema test using Rails' own models/tables; test names
  match Rails verbatim if a corresponding Rails test exists.
