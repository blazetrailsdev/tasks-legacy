---
title: "Window#frame/rows/range return this where Rails returns the frame node"
status: draft
updated: 2026-08-11
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "arel"
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

# `Window#frame`/`rows`/`range` return `this` where Rails returns the frame node

## Context

Surfaced while converging `Window#order`/`partition` in PR #6361 (the
`String → SqlLiteral` coercion that had been dropped from `window.rb:14-27`).
The three neighbouring methods carry a separate, unconverged divergence.

**Rails** (`vendor/rails/activerecord/lib/arel/nodes/window.rb:30-48`):

```ruby
def frame(expr)
  @framing = expr
end

def rows(expr = nil)
  if @framing
    Rows.new(expr)
  else
    frame(Rows.new(expr))
  end
end

def range(expr = nil)
  if @framing
    Range.new(expr)
  else
    frame(Range.new(expr))
  end
end
```

Ruby's assignment expression makes `frame` return the node it just stored, so
BOTH arms of `rows`/`range` evaluate to a frame node — never the Window.

**trails** (`packages/arel/src/nodes/window.ts:34-53`) returns `this` from
`frame`, and returns `this` from the `else` arm of `rows`/`range` after calling
`frame` for its side effect only. The return type is spelled `Rows | this` /
`Range | this`, so the two arms disagree and every caller of the un-framed arm
gets the Window instead of the frame node it can chain `preceding`/`following`
off — which is how `over(...)` window frames are built.

## Converged shape

`frame(expr)` returns the stored node (`Node`, not `this`); `rows`/`range`
return `Rows` / `Range` from both arms, matching window.rb:30-48. Audit callers
that currently rely on the `this` return for chaining.

## Acceptance criteria

1. `frame`, `rows` and `range` return what window.rb:30-48 returns, from both
   arms; the `Rows | this` / `Range | this` union collapses to the node type.
2. Callers relying on the Window return are updated.
3. `pnpm vitest run packages/arel` green; generated SQL byte-identical for the
   already-framed path.
