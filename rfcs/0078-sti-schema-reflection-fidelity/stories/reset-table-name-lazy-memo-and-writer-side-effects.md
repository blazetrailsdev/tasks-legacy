---
title: "Fold resolveTableName back into Rails' table_name / reset_table_name / compute_table_name split"
status: claimed
updated: 2026-08-24
rfc: "0078-sti-schema-reflection-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 140
priority: 2
pr: null
claim: "2026-08-24T16:20:09Z"
assignee: "sync-reflection-needs-explicit-warm-for-fake-adapter"
blocked-by: null
closed-reason: null
---

# Fold `resolveTableName` back into Rails' `table_name` / `reset_table_name` / `compute_table_name` split

## Context

Surfaced by #6821 (`abstract-class-table-name-and-load-schema-guard`), which had
to put `reset_table_name`'s arms inside `resolveTableName` because that is what
the `tableName` reader actually routes through.

Rails has three methods with three jobs
(`vendor/rails/activerecord/lib/active_record/model_schema.rb`):

```ruby
def table_name                                    # :260-263
  reset_table_name unless defined?(@table_name)
  @table_name
end

def table_name=(value)                            # :270-282
  ...
  reset_column_information if connected?
  @table_name = value
  @arel_table = nil
  @sequence_name = nil unless @explicit_sequence_name
  @predicate_builder = nil
end

def reset_table_name                              # :290-300
  self.table_name = if self == Base
    nil
  elsif abstract_class?
    superclass.table_name
  elsif superclass.abstract_class?
    superclass.table_name || compute_table_name
  else
    compute_table_name
  end
end
```

The reader is lazy and MEMOIZES into `@table_name`, and it reaches its value by
ASSIGNING through `table_name=`, so first read also clears `@arel_table`,
`@sequence_name` and `@predicate_builder`.

trails splits this differently
(`packages/activerecord/src/model-schema.ts`):

- `tableName()` (:1538) returns `resolveTableName.call(this)`.
- `resolveTableName` (:80) is the real reader: it early-returns `_tableName`,
  then carries `self == Base` and `abstract_class?` (added by #6821), then the
  `baseClass` descent, then `compute_table_name`'s prefix/suffix/contained join.
  It does NOT memoize and never assigns through the writer.
- `resetTableName` (:614) nils `_tableName`, re-checks `self == Base`, and
  delegates to `resolveTableName`.

So the arms are duplicated across two functions, every `tableName` read
recomputes the join, and no read ever runs the writer's cache invalidation.
Rails' `superclass.abstract_class?` arm has no named counterpart at all — trails
reaches the same answer through `isBaseClass` / `baseClass`.

## Converged shape

- `tableName` reads `if (!hasOwn _tableName) resetTableName(); return _tableName`
  — lazy, memoizing, per Rails' `defined?(@table_name)` guard.
- `resetTableName` carries all four arms verbatim and reaches them by assigning
  through the `tableName` writer, so the `_arelTable` / `_sequenceName` /
  `_predicateBuilder` clears happen where Rails has them.
- `resolveTableName` collapses to `compute_table_name` — prefix, contained
  prefix, undecorated name, suffix — and loses the `Base` / `abstract_class?` /
  `baseClass` arms.

Watch the memoization: Ruby class-instance variables are not inherited, so the
`hasOwn` check is load-bearing (`ownSchemaMemo` in the same file is the existing
idiom). A plain prototype read hands a subclass on another table the base's name.

## Acceptance criteria

- [ ] `resolveTableName` contains no `self == Base`, `abstract_class?` or
      `baseClass` arm; those live once in `resetTableName`.
- [ ] A first `tableName` read memoizes and clears the writer's three caches.
- [ ] `superclass.abstract_class?` is a named arm, not an emergent result of the
      `baseClass` descent.
- [ ] `AbstractStiPost.tableName === "posts"`, an abstract class extending `Base`
      still raises `TableNotSpecified`, and `base.test.ts`'s
      "table name based on model name" family stays green.
