---
title: "InsertAll defers its constructor tail (@returning, unique_by, ensure_valid_options) to execute"
status: done
updated: 2026-08-25
rfc: "0112-one-rails-thing-n-trails-things"
cluster: split-stores
packages: []
deps: []
deps-rfc: []
est-loc: 140
pr: 7053
claim: "2026-08-25T16:34:34Z"
assignee: "converge-association-relation-through-scope-onto-scoping"
blocked-by: null
closed-reason: null
---

## Context

Rails' `InsertAll#initialize` (`insert_all.rb:20-45`) finishes its whole setup
synchronously, in this order (`:38-45`):

```ruby
@returning = (connection.supports_insert_returning? ? primary_keys : false) if @returning.nil?
@returning = false if @returning == []
@unique_by = find_unique_index_for(@unique_by)
configure_on_duplicate_update_logic
ensure_valid_options_for_connection!
```

trails' constructor cannot: `primary_keys` is
`schema_cache.primary_keys(table_name)` (async here), `find_unique_index_for`
reads `cache.indexes()`, and since #6226 (RFC 0072
`make-version-gated-predicates-async`) `supports_insert_returning?` is async too.
So `_populateUpdatableColumns` re-runs that constructor tail on the first
`execute()`, with `_schemaCachePrimaryKeys === undefined` and
`_uniqueByResolved` acting as run-once guards, and `returning` left `undefined`
(Ruby's nil sentinel, `insert_all.rb:38`) until then.

The ordering matches Rails and the guards give constructor-once semantics, but
the work is still not where Rails puts it: a caller that constructs an
`InsertAll` and inspects `returning`/`uniqueBy` before `execute()` sees
unresolved state, and `ensure_valid_options_for_connection!` raises later than
Rails raises it.

## Converged shape

The constructor completes its own setup, so `InsertAll.new` is fully resolved
the way Rails' is. Needs the async surface pushed to the caller — most plausibly
an async factory (`InsertAll.build`/`execute` doing the awaits before
constructing), keeping the Rails-named constructor body a pure assignment of
already-resolved values rather than a deferred tail.

## Acceptance criteria

- [ ] `returning`, `uniqueBy` and the `ensure_valid_options_for_connection!`
      raise are all settled before an `InsertAll` is observable, matching
      `insert_all.rb:38-45`.
- [ ] `_returningDefaulted` / `_uniqueByResolved` / the deferred block in
      `_populateUpdatableColumns` are removed, not merely relocated.
- [ ] `insert-all.test.ts` and `insert-all.trails.test.ts` stay green; the
      Rails-matched test names are unchanged.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
