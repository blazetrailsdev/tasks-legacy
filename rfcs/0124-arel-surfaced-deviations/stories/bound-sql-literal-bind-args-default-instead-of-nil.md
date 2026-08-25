---
title: "BoundSqlLiteral defaults its bind collections where Rails requires them and passes nil"
status: ready
updated: 2026-08-25
rfc: "0124-arel-surfaced-deviations"
cluster: null
packages: []
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

Surfaced in #7043 while porting `Arel.sql`'s bound arm (RFC 0124,
`arel-sql-drops-the-bound-sql-literal-arm`).

Rails' `BoundSqlLiteral#initialize` takes all three arguments **required** and
handles `nil` for either bind collection explicitly
(`vendor/rails/activerecord/lib/arel/nodes/bound_sql_literal.rb:8-10`):

```ruby
def initialize(sql_with_placeholders, positional_binds, named_binds)
  has_positional = !(positional_binds.nil? || positional_binds.empty?)
  has_named = !(named_binds.nil? || named_binds.empty?)
```

and its callers pass `nil` for the side they are not using
(`activerecord/lib/active_record/relation/query_methods.rb:1696` —
`BoundSqlLiteral.new("(#{statement})", nil, bound_values)` — and `:1716`,
the positional mirror).

trails gives both parameters defaults
(`packages/arel/src/nodes/bound-sql-literal.ts:26-30`:
`positionalBinds: unknown[] = []`, `namedBinds: Record<string, unknown> = {}`)
and its `validate()` tests only `.length > 0` / `Object.keys().length > 0`,
never a nil. The trails callers correspondingly pass `[]` / `{}`
(`packages/activerecord/src/relation/query-methods.ts:2312` and `:2330`).

Behaviour matches today — an empty collection and `nil` take the same branch —
but the arity and the argument shape both differ from Rails, and the defaults
mean a caller can omit an argument Rails requires.

## Converged shape

Make `positionalBinds` and `namedBinds` required and admit `null`, spelling the
guard as Rails does (`positionalBinds == null || positionalBinds.length === 0`),
and pass `null` from `buildNamedBoundSqlLiteral` / `buildBoundSqlLiteral` for
the unused side, matching `query_methods.rb:1696` and `:1716`. `Arel.sql`'s
bound arm keeps passing both collections, as `arel.rb:56` does.

## Acceptance criteria

- [ ] `BoundSqlLiteral`'s constructor arity and nil handling match
      `bound_sql_literal.rb:8-10`.
- [ ] The two `relation/query-methods.ts` call sites pass what
      `query_methods.rb:1696`/`:1716` pass.
- [ ] `pnpm parity:api` arity delta non-negative; `parity:api:calls:args`
      clean with no new baseline row; arel + activerecord suites green.
