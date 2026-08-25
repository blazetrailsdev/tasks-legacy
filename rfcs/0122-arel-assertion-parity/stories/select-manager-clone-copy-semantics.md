---
title: "SelectManager has no clone, so the two Rails clone tests can't be mirrored"
status: ready
updated: 2026-08-25
rfc: "0122-arel-assertion-parity"
cluster: null
packages: []
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

`select_manager_test.rb`'s `describe "clone"` block
(`vendor/rails/activerecord/test/cases/arel/select_manager_test.rb:147-166`) is
the last pair of assertion mismatches left in
`packages/arel/src/select-manager.test.ts` after RFC 0122's
`select-manager-assertion-parity` (both rows: `creates new cores`,
`makes updates to the correct copy`).

Both Rails tests call `mgr.clone` and assert with `wont_equal` / `must_equal`
on the two managers' SQL:

```ruby
it "creates new cores" do
  table = Table.new :users, as: "foo"
  mgr = table.from
  m2 = mgr.clone
  m2.project "foo"
  _(mgr.to_sql).wont_equal m2.to_sql
end
```

trails has no `TreeManager#clone` — Rails gets one from Ruby's `clone` plus
`TreeManager#initialize_copy` (`vendor/rails/activerecord/lib/arel/tree_manager.rb`),
which deep-copies `@ast`. The trails tests therefore assert something else
entirely (core counts, substring checks) under the Rails test names, which is
why they were left unconverged rather than reworded.

## Acceptance criteria

- `TreeManager` gets Rails' copy semantics (the `initialize_copy` port) under a
  name a Rails dev recognises, so a manager can be copied with an independent
  AST.
- Both `clone` tests in `packages/arel/src/select-manager.test.ts` mirror the
  Ruby bodies above; no test name changes.
- `scripts/test-compare/assertion-mismatch-mark.json`'s arel entry is tightened
  by what this converges; `pnpm parity:test:assertions` green.
