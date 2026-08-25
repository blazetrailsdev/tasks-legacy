---
title: "Converge appendConstraints onto Rails' in-place join mutation"
status: draft
updated: 2026-07-30
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 60
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Rails' `append_constraints` **mutates the join node in place** and returns
nothing (`activerecord/lib/active_record/associations/join_dependency/join_association.rb:92-100`):

```ruby
def append_constraints(join, constraints)
  if join.is_a?(Arel::Nodes::StringJoin)
    join_string = Arel::Nodes::And.new(constraints.unshift join.left)
    join.left = join_string
  else
    right = join.right
    right.expr = Arel::Nodes::And.new(constraints.unshift right.expr)
  end
end
```

trails' port is **pure** — it builds and returns a replacement node
(`packages/activerecord/src/associations/join-dependency/join-association.ts:288`),
with the stated reason "Arel join node fields are readonly". That forces two
call sites to write the result back into the array
(`join-association.ts:207` and `:212`):

```ts
sources[lastIdx] = appendConstraints(sources[lastIdx], others) ?? sources[lastIdx];
```

Rails needs no write-back at all: it concats into its own local `joins` array
and then mutates `joins.last` in place.

This surfaced in PR #5631, which made `SelectManager#joinSources()` return
Arel's **live** `source.right` (matching `arel/select_manager.rb:244-246`).
The write-back at `:207` would then have mutated the manager's live array, so
that call site had to take a defensive copy. The copy is a workaround for the
pure-vs-mutating divergence, not a Rails behavior.

Note the Arel-side premise is also worth checking: in Rails, `Arel::Nodes::On#expr`
and `StringJoin#left` are plain writable accessors, so "fields are readonly" is
a trails-side constraint, not a Rails one.

## Acceptance criteria

- `appendConstraints` mutates the join node in place and returns `void`,
  matching `join_association.rb:92-100`.
- The Arel join/`On` node fields it assigns are writable (mirroring Rails'
  `attr_accessor`), or the story documents why they cannot be.
- Both call sites drop the array write-back, matching Rails' shape.
- The defensive copy added at `join-association.ts:198` in #5631 is removed
  once the in-place mutation no longer targets Arel's live `source.right`.
- No SQL changes: existing `associations/` and `relation/` suites stay green.
