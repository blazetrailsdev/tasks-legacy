---
title: '_delete_record issues an unnamed DELETE where Rails passes "#{self} Destroy"'
status: draft
updated: 2026-08-13
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 40
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`ClassMethods#_delete_record` issues its DELETE with no `name`, so the
`sql.active_record` payload carries a null name where Rails names the query:

    # vendor/rails/activerecord/lib/active_record/persistence.rb:295-297
    with_connection do |c|
      c.delete(dm, "#{self} Destroy")
    end

trails (`packages/activerecord/src/persistence.ts`, `_deleteRecord`):

    if (typeof adapter.delete === "function") {
      return adapter.delete(dm);
    }

`AbstractAdapter#delete(arel, name = null, binds = [])` already takes the name
and threads it to `execDelete`, so the argument is simply dropped.

Found while fixing the identical omission in `_update_record`, which PR #6473
surfaced: `c.update(um, "#{self} Update")` (persistence.rb:277-279) was also
unnamed, and routing the save path through `_update_row` made it observable as
`instrumentation.test.ts:46` (`expected null to be "Book Update"`) on all three
adapters. The UPDATE half is fixed; the DELETE half is not, and is currently
unobserved only because `instrumentation.test.ts`'s `"Book Destroy"` assertion
is satisfied by a different code path.

## Converged shape

    return adapter.delete(dm, `${(this as any).name} Destroy`);

exactly as the `_updateRecord` sibling now reads.

## Acceptance criteria

- [ ] `_deleteRecord` passes `"#{self} Destroy"` as `delete`'s `name`.
- [ ] A `sql.active_record` subscriber sees `"Book Destroy"` for the DELETE
      `_deleteRecord` issues (not only for whichever path satisfies the
      existing assertion) — add the arm to `instrumentation.test.ts` if the
      current one does not exercise `_deleteRecord`.
