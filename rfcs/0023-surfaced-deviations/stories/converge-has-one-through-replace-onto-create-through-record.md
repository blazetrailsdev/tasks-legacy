---
title: "Converge HasOneThroughAssociation#replace onto create_through_record"
status: draft
updated: 2026-08-20
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 220
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while landing `wave-4d-associations-residue-part-5` (PR #6763). The
call-set baseline row

    associations/has-one-through-association.ts  replace  create_through_record

could not be converged in that PR and now carries a verified per-site reason
instead. It is debt, not a settled decision.

Rails (`activerecord/lib/active_record/associations/has_one_through_association.rb:9-13`):

    def replace(record, save = true)
      create_through_record(record, save)
      self.target = record
    end

`create_through_record` (`:15-40`) builds/creates/updates/destroys the join
model, so in trails it is necessarily `async`. `replace` cannot be: it is
reached from the synchronous `writer` / `setNewRecord` path that RFC 0087
requires to stay sync (`project_rfc0087_sync_assign_attributes_must_not_reach_association_writers`).
trails therefore queues a `_pendingReplace` marker
(`packages/activerecord/src/associations/has-one-through-association.ts:310-345`)
that `persistReplace` / `autosaveHasOne` drains a frame later by awaiting
`createThroughRecord`.

Note `converge-has-one-through-persist-onto-autosave` (0023, done, PR #4493)
deliberately left the through association on this deferred path when the
**direct** `HasOneAssociation` retired its `_pendingReplace` machinery
(PR #4489). That decision is what this story revisits.

## Converged shape

`replace(record, save = true)` calls `createThroughRecord(record, save)` and
then sets `this.target = record`, in that order, with no `_pendingReplace`
marker and no `persistReplace` drain — matching the direct
`HasOneAssociation` convergence PR #4489 already achieved.

The blocker to resolve first: whether the sync `writer` / `setNewRecord` entry
can hand this association an awaitable seam without violating RFC 0087's rule
that sync `assignAttributes` must not reach association writers. If it
genuinely cannot, `pnpm tasks block` with that finding rather than re-reasoning
the baseline row.

## Acceptance criteria

- [ ] `replace` makes the `createThroughRecord` call Rails makes, at the Rails
      call site, or the story is blocked with the specific RFC 0087 conflict.
- [ ] `_pendingReplace` / `persistReplace` / `flushPendingReplaces` retired for
      `HasOneThroughAssociation`, or explicitly scoped out with a reason.
- [ ] The baseline row above is deleted by hand via `serializeBaseline` and its
      mark shard tightened. No `--write`, no reseed.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.
