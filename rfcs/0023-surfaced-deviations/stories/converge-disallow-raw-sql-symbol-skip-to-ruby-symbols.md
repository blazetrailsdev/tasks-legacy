---
title: "converge-disallow-raw-sql-symbol-skip-to-ruby-symbols"
status: draft
updated: 2026-08-16
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

Surfaced converging `preprocess_order_args` in #6607.
`disallowRawSqlBang` (`packages/activerecord/src/sanitization.ts:133-142`)
skips Symbols with:

```ts
if (typeof arg === "symbol") continue;
```

Rails (activerecord/lib/active_record/sanitization.rb:186) is
`next if arg.is_a?(Symbol) || Arel.arel_node?(arg) || permit.match?(arg.to_s.strip)`.

A trails Ruby Symbol is a `":name"` **string**, never a JS `Symbol` (see
CLAUDE.md, "A Ruby Symbol is a JS string"), so that arm is dead: every Symbol
argument falls through to the matcher, where the leading colon fails it. Callers
paper over it — `preprocessOrderArgs` (`relation/query-methods.ts`) pre-maps
`isRubySymbol(k) ? symbolToName(k) : k` over the flattened args purely so the
descriptions rather than the colons reach the matcher. Rails does no such
mapping; `disallow_raw_sql!` simply skips them.

Converging the skip to `isRubySymbol(arg)` removes that pre-map and makes the
`pluck` call site (calculations.rb:313) behave the same way.

## Acceptance criteria

- [ ] `disallowRawSqlBang`'s Symbol arm tests `isRubySymbol(arg)`, matching
      sanitization.rb:186; the dead `typeof arg === "symbol"` test goes or is
      folded into it.
- [ ] `preprocessOrderArgs`' `symbolToName` pre-map over `flattenedArgs` is
      deleted — `disallow_raw_sql!` does the skipping, as in Rails.
- [ ] `unsafe-raw-sql.test.ts` and `with.test.ts` stay green; the `order(foo:
:asc)` and `pluck(:id)` symbol paths are covered.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.
