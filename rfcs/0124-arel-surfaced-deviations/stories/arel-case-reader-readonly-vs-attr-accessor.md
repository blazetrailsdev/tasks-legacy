---
title: "Arel::Nodes::Case#case is readonly where Rails has attr_accessor"
status: ready
updated: 2026-08-25
rfc: "0124-arel-surfaced-deviations"
cluster: null
packages:
  - "arel"
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

`Arel::Nodes::Case` declares all three readers writable:

```ruby
# activerecord/lib/arel/nodes/case.rb:5-6
class Case < Arel::Nodes::NodeExpression
  attr_accessor :case, :conditions, :default
```

trails' `packages/arel/src/nodes/case.ts` declares `readonly case: Node | null`.
`conditions` was `readonly` too until PR #7009 made it writable so the mirrored
`visitors/dot_test.rb` port could assign it the way Rails' test does
(`dot_test.rb:115-118`); `case` was left alone because nothing in that PR
needed it.

`readonly` on a reader Rails exposes through `attr_accessor` is a silent
divergence: any port that assigns `node.case` has to reach through a cast, and
the cast is what hides the mismatch.

## Acceptance criteria

- [ ] `Case#case` is writable, matching `attr_accessor :case` (`case.rb:6`).
- [ ] Sweep the rest of `packages/arel/src/nodes/` for readers declared
      `readonly` where the Ruby node uses `attr_accessor` rather than
      `attr_reader`, and converge those too; list what was found.
- [ ] `pnpm parity:api` delta non-negative; arel suite green.
