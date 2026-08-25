---
title: "Restore weakened arel ported test bodies"
status: done
updated: 2026-08-25
rfc: "0124-arel-surfaced-deviations"
cluster: null
packages:
  - "arel"
deps: []
deps-rfc: []
est-loc: 80
priority: null
pr: 7054
claim: "2026-08-25T16:56:36Z"
assignee: "arel-star-is-a-shared-const-not-a-per-call-method"
blocked-by: null
closed-reason: null
---

## Context

The weakened-assertion sweeps under this RFC covered activerecord; the **arel**
package was never swept, and a weakened body there hid two real visitor bugs.

Rails' `test_join_sources`
(`vendor/rails/activerecord/test/cases/arel/select_manager_test.rb:7-10`)
mutates the array returned by `join_sources` and asserts the resulting SQL:

```ruby
def test_join_sources
  manager = Arel::SelectManager.new
  manager.join_sources << Arel::Nodes::StringJoin.new(Nodes.build_quoted("foo"))
  assert_equal "SELECT FROM 'foo'", manager.to_sql
end
```

trails' port asserted only `expect(mgr.joinSources()).toEqual([])` — it never
exercised the mutation or the SQL. PR #5631 restored Rails' body, and it failed
immediately, exposing two divergences in `packages/arel/src/visitors/to-sql.ts`:

- `visit_Arel_Nodes_SelectCore` gated `" FROM "` on `source.left` instead of
  Rails' `!o.source.empty?` (`arel/visitors/to_sql.rb:157-160`), so a source
  carrying only joins emitted no `FROM` at all.
- `visit_Arel_Nodes_JoinSource` prefixed every join with a space instead of
  emitting the separator only when a left exists and then `inject_join`ing
  (`to_sql.rb:509-516`).

Both are fixed in #5631, along with a third in the MySQL visitor that the same
shape exposed. The point of this story is that **one weakened arel test body
was concealing three separate visitor divergences**, so the rest of the arel
suite is worth the same sweep.

## Acceptance criteria

- Sweep ported arel test bodies against
  `vendor/rails/activerecord/test/cases/arel/` for bodies that assert weaker
  facts than Rails' (dropped mutations, `toEqual([])` where Rails asserts SQL,
  missing `assert_equal` on the compiled string).
- Restore each to Rails' body verbatim; test names stay unchanged.
- Fix — do not re-weaken — any implementation divergence a restored body
  exposes; file separately if a fix exceeds this story's budget.
- Start from `select_manager_test.rb` and the `visitors/` tests, which are the
  highest-yield files based on #5631.
- `pnpm parity:test` delta non-negative and `pnpm vitest run packages/arel`
  green; report the assertion-mismatch counts for arel before and after, since a
  restored body makes previously-uncomparable assertions comparable and the mark
  in `scripts/test-compare/assertion-mismatch-mark.json` is only-shrink.
