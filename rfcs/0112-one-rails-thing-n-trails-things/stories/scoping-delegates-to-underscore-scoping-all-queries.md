---
title: "scoping should delegate to _scoping, and _scoping should handle all_queries"
status: done
updated: 2026-08-22
rfc: "0112-one-rails-thing-n-trails-things"
cluster: duplicate-bodies
deps: []
deps-rfc: []
est-loc: 60
pr: 6811
claim: "2026-08-21T11:40:36Z"
assignee: "hash-config-primary-resolves-via-global-configurations"
blocked-by: null
closed-reason: null
---

## Context

Rails' public `Relation#scoping` ends by delegating to the private
`_scoping` (`vendor/rails/activerecord/lib/active_record/relation.rb:541-550`):

```ruby
_scoping(self, registry, all_queries, &block)
```

Our port at `packages/activerecord/src/relation.ts:6600-6641` instead
**inlines** the save/set/restore sequence, duplicating what `_scoping`
(`relation.ts:7093-7101`) already does.

The two have since diverged: Rails' `_scoping`
(`relation.rb:1365-1379`) also handles the `all_queries` arm, saving and
restoring `global_current_scope`. Ours takes only `(scope, registry, fn)`
and handles `current_scope` alone — the global-scope handling lives only
in the inlined copy inside `scoping`.

Surfaced during review of #4978, which introduced the `_execScope` ->
`_scoping` call path and put the two implementations side by side. Not
caused by that PR; pre-existing.

## Acceptance criteria

- `scoping` delegates to `_scoping` rather than inlining it.
- `_scoping` accepts `all_queries` and saves/restores
  `global_current_scope`, mirroring `relation.rb:1365-1379`.
- `_execScope` continues to call `_scoping(null, registry, ...)` so the
  `already_in_scope?` / `global_scope?` checks in the public `scoping`
  stay bypassed, per `relation.rb:552-557`.
- `scoping/` suites plus `relation/unscope-coverage.test.ts` stay green.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
