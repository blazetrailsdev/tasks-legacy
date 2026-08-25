---
title: "in_order_of records references_values on the spawn, not the receiver as Rails does"
status: draft
updated: 2026-08-15
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

# `in_order_of` records `references_values` on the spawn, where Rails records them on the receiver

## Context

Surfaced converging the `in_order_of -> column_references` call row in PR 6563. The row is converged; this ordering difference is what the call site
comment there flags.

Rails `QueryMethods#in_order_of`,
`activerecord/lib/active_record/relation/query_methods.rb:717-737`:

    def in_order_of(column, values, filter: true)
      model.disallow_raw_sql!([column], permit: ...)
      return spawn.none! if values.empty?

      references = column_references([column])
      self.references_values |= references unless references.empty?
      ...
      scope = spawn.order!(build_case_for_value_position(arel_column, values, filter: filter))

`self.references_values |= references` mutates the RECEIVER, before
`spawn`. The spawn then copies the mutated values, so the reference lands on
both the receiver and the returned relation.

trails clones up front (`const rel = this._clone()`) and applies the
references to `rel` only, so the receiver does not keep them. For

    const base = Post.where(...)
    base.inOrderOf("comments.body", [...])

Rails leaves `base.references_values` containing `comments`; trails leaves it
empty. A later `base.includes(:comments)` would be promoted to `eager_load`
in Rails and not in trails.

## Converged shape

Apply `columnReferences` to `this` before cloning, matching
query_methods.rb:721-722 — or establish that trails' clone-first spawn
ordering is a deliberate, documented repo-wide policy and cite it here. The
former is the convergence; the latter needs a real citation, not a
preference.

Rails' receiver mutation is surprising but intentional and observable, so
this is a behaviour difference, not a style one.

## Acceptance criteria

- [ ] `inOrderOf` records references where Rails records them, verified
      against query_methods.rb:717-737.
- [ ] A test pins the receiver's `referencesValues` after an `inOrderOf`
      call, and fails on the pre-fix baseline.
- [ ] `relation/field-ordered-values.test.ts` stays green.
- [ ] All three adapter lanes green.
