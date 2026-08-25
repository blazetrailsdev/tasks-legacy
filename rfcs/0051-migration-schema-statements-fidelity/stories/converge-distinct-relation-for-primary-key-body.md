---
title: "Converge distinct_relation_for_primary_key's body onto schema_statements.rb:1429-1452"
status: in-progress
updated: 2026-08-25
rfc: "0051-migration-schema-statements-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 90
priority: 37
pr: 7032
claim: "2026-08-25T12:58:54Z"
assignee: "split-model-mixin-surface-to-active-model-model"
blocked-by: null
closed-reason: null
---

## Context

`distinctRelationForPrimaryKey`
(`packages/activerecord/src/connection-adapters/abstract/schema-statements.ts:1729`)
gained its first production caller in PR #6634, which routed
`apply_join_dependency`'s limit/offset-over-a-collection branch through it. With
the method now on a hot path, three divergences from
`vendor/rails/activerecord/lib/active_record/connection_adapters/abstract/schema_statements.rb:1429-1452`
are worth converging.

Rails:

```ruby
def distinct_relation_for_primary_key(relation) # :nodoc:
  primary_key_columns = Array(relation.primary_key).map do |column|
    visitor.compile(relation.table[column])
  end

  values = columns_for_distinct(primary_key_columns, relation.order_values)

  limited = relation.reselect(values).distinct!
  limited_ids = select_rows(limited.arel, "SQL").map do |results|
    results.last(Array(relation.primary_key).length)
  end

  if limited_ids.empty?
    relation.none!
  else
    relation.where!(**Array(relation.primary_key).zip(limited_ids.transpose).to_h)
  end

  relation.limit_value = relation.offset_value = nil
end
```

Divergences:

1. **`if (!pk) return;` early guard (`:1742`) has no Rails counterpart.** Rails
   has no nil-pk guard: `Array(nil)` is `[]`, so a pk-less relation compiles an
   empty `columns_for_distinct` projection and proceeds. trails returns early,
   leaving `limit_value`/`offset_value` SET where Rails clears them
   unconditionally (`schema_statements.rb:1451`). A pk-less eager relation with
   a limit therefore keeps a limit Rails would have dropped.
2. **Primary-key columns are built by string interpolation, not the visitor.**
   trails spells `${quoteTableName(table.name)}.${quoteColumnName(c)}`
   (`:1752-1757`); Rails is `visitor.compile(relation.table[column])`
   (`schema_statements.rb:1431`). The Arel path is what makes a `TableAlias`
   receiver, a schema-qualified name, or an adapter-specific quoting rule come
   out right.
3. **Optional-member guards around a duck-typed argument.** trails declares the
   parameter as a structural type with every member optional and guards
   `if (limited.reselect)` / `relation.whereBang?.(...)` / `typeof
this.quoteColumnName === "function"`. Rails calls `reselect`, `distinct!`,
   `none!` and `where!` unguarded — the argument is always a `Relation`. The
   guards let a genuine wiring break pass silently instead of raising.

Note the `Promise<void>` return (Rails returns `relation`) is NOT in scope: it
is a real TypeScript shortcoming — a trails `Relation` is thenable, so it cannot
be returned through a `Promise` without being executed — and Rails' own rewrites
are all in-place mutations of the argument, so the caller holds the same object.
That deviation is justified at the method and should stay.

## Acceptance criteria

- The nil-pk early return is removed; a pk-less relation follows Rails' path and
  gets `limitValue`/`offsetValue` cleared like every other.
- Primary-key columns are compiled through the Arel visitor over
  `relation.table[column]`, matching `schema_statements.rb:1431`.
- The parameter is typed as the `Relation` surface it actually receives and the
  optional-member guards are dropped, so `reselect` / `distinctBang` / `noneBang`
  / `whereBang` are called unguarded as Rails calls them.
- `schema-statements-privates.test.ts`'s `distinctRelationForPrimaryKey` block
  covers the pk-less arm.
- `pnpm parity:api:calls` / `:args` clean; `parity:api` / `parity:test` deltas
  non-negative.
