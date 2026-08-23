---
title: "EXPLAIN's options join emits a Ruby Symbol's colon, which Symbol#to_s drops"
status: ready
updated: 2026-08-23
rfc: "0112-one-rails-thing-n-trails-things"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 130
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Both adapters now render `Relation#explain`'s options with Rails' bare join
(PR #6888 for MySQL, #6890 for PG), matching
`mysql/database_statements.rb:39` and
`postgresql/database_statements.rb:99`:

```ruby
"EXPLAIN #{options.join(" ").upcase}"
"EXPLAIN (#{options.join(", ").upcase})"
```

Ruby's `Array#join` calls `to_s` on each element, and `Symbol#to_s` DROPS the
leading colon, so Rails' documented Symbol spelling works:

```ruby
[:analyze, :buffers].join(", ").upcase   #=> "ANALYZE, BUFFERS"
```

trails spells a Ruby Symbol as a colon-prefixed string (CLAUDE.md, "A Ruby
Symbol is a JS string"), and JS `Array#join` has no `to_s` hook, so the same
call emits the colon:

```js
[":analyze", ":buffers"].join(", ").toUpperCase(); //=> ":ANALYZE, :BUFFERS"
```

`EXPLAIN (:ANALYZE, :BUFFERS)` is not valid SQL on any adapter. The bug is
pre-existing — the old per-option `map` upcased a string element without
stripping a colon either — but the single `join` is now the one place to fix
it.

Rails exercises exactly this arm:
`activerecord/test/cases/adapters/postgresql/explain_test.rb:23-27`

```ruby
def test_explain_with_options_as_symbols
  explain = Author.where(id: 1).explain(:analyze, :buffers).inspect
  assert_match %r(EXPLAIN \(ANALYZE, BUFFERS\) SELECT ...), explain
```

Our port of that test —
`packages/activerecord/src/adapters/postgresql/explain.test.ts:96-102` —
passes **no options at all** and asserts only that the plan mentions the table,
so it carries the Rails test name without exercising the Symbol arm. That is
why the divergence has stayed invisible.

## Converged shape

The join applies `Symbol#to_s` per element, which is a `.slice(1)` on a leading
colon — the trails spelling already used elsewhere for the same conversion.
Keep it inside `buildExplainClause` in each adapter, so both bodies stay the
single Rails expression they are now rather than gaining a validator:

```ts
`EXPLAIN ${options.map(symbolToS).join(" ").toUpperCase()}``EXPLAIN (${options.map(symbolToS).join(", ").toUpperCase()})`;
```

A plain String option (`"analyze"`, `"format json"`) is untouched, so Rails'
`test_explain_with_options_as_strings` keeps passing.

Check whether any other ported body joins a Symbol array the same way; if so,
file those separately rather than widening this story.

## Acceptance criteria

- [ ] `Post.all().explain(":analyze", ":buffers")` renders
      `EXPLAIN (ANALYZE, BUFFERS)` on PG and `EXPLAIN ANALYZE BUFFERS` on MySQL.
- [ ] `explain with options as symbols`
      (`postgresql/explain.test.ts:96`) actually passes `":analyze"`/`":buffers"`
      and asserts Rails' `EXPLAIN (ANALYZE, BUFFERS) SELECT ...` regex
      (`postgresql/explain_test.rb:23-27`). No test rename.
- [ ] The String spelling is unaffected —
      `test_explain_with_options_as_strings` (`explain_test.rb:29-33`) stays
      green.
- [ ] `pnpm parity:test:assertions` green.
