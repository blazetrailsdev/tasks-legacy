---
title: "Arel DeleteManager#group / UpdateManager#group drop Rails' Symbol arm"
status: ready
updated: 2026-08-25
rfc: "0113-branch-and-guard-parity"
cluster: missing-arm
packages: []
deps: []
deps-rfc: []
est-loc: 70
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`Arel::DeleteManager#group` and `Arel::UpdateManager#group` each carry a Symbol
arm that trails' port drops.

Rails (`activerecord/lib/arel/delete_manager.rb:16-24`, identical body at
`activerecord/lib/arel/update_manager.rb:33-41`):

```ruby
def group(columns)
  columns.each do |column|
    column = Nodes::SqlLiteral.new(column) if String === column
    column = Nodes::SqlLiteral.new(column.to_s) if Symbol === column

    @ast.groups.push Nodes::Group.new column
  end

  self
end
```

trails today (`packages/arel/src/delete-manager.ts:45-55`,
`packages/arel/src/update-manager.ts:85-95`) has the String arm and no Symbol
arm — a Symbol reaches `new Group(column)` unwrapped.

Rails' own test exercises the arm directly:
`activerecord/test/cases/arel/update_manager_test.rb:64-73`,
`it "adds columns to the AST when group value is a Symbol"` →
`update_manager.group([:"posts.id"])`, asserting
`group_ast.expr == "posts.id"`. trails' mirror of that test
(`packages/arel/src/update-manager.test.ts`, same name) passes an
`Attribute` rather than a Symbol, so it does not currently cover the arm it is
named for.

Per CLAUDE.md a Ruby Symbol is a JS string, so the two Rails arms collapse
differently here than they look — work out whether the Symbol arm is reachable
at all in the trails value space BEFORE porting it. If it is genuinely
unreachable, that conclusion belongs in a `SKIP_GROUPS`-style recorded reason,
not in silence.

Noted while fixing the `group` signature on these two managers in PR #6350
(RFC 0096 arel naming burndown); out of scope there, which was a rename-only PR.

## Converged shape

Port both arms in both files, and make the trails
`"adds columns to the AST when group value is a Symbol"` test actually exercise
the Symbol arm the way `update_manager_test.rb:65` does — WITHOUT renaming the
test (it is a `parity:test` match key).

## Acceptance criteria

1. `DeleteManager#group` and `UpdateManager#group` carry both the String and the
   Symbol arm, in Rails' order, or the Symbol arm's unreachability is recorded
   with a reason.
2. The existing `...when group value is a Symbol` test exercises the Symbol arm;
   the test name is unchanged and the test FAILS on baseline.
3. `pnpm parity:api:calls` / `pnpm parity:api:calls:args` stay green.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0113-branch-and-guard-parity`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
