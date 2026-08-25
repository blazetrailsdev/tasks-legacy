---
title: "visit_CreateIndexDefinition uppercases index.type where Rails emits it verbatim"
status: ready
updated: 2026-08-25
rfc: "0051-migration-schema-statements-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 30
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while inlining the `<index name> ON <table>` fragment in #7044
(`sqlite-attached-schema-notion-has-no-rails-counterpart`), which restored
`visit_CreateIndexDefinition` to a no-extraction mirror. This arm sits two
lines above that fragment and was left alone as out of scope.

Rails emits the index type **verbatim**:

```ruby
# activerecord/lib/active_record/connection_adapters/abstract/schema_creation.rb:114
sql << index.type if index.type
```

`sql` is an array joined with `" "` at `:122`, so the value reaches the SQL
exactly as stored — `index.type` is set from the caller's `type:` option and is
lower-case in every Rails path that sets it.

trails uppercases it:

```ts
// packages/activerecord/src/connection-adapters/abstract/schema-creation.ts:311
if (index.type) parts.push(index.type.toUpperCase());
```

This is an observable SQL-text divergence, not a cosmetic one. It is reachable
on MySQL, whose `add_index` accepts `type: :fulltext` and `type: :spatial`, so
trails emits `CREATE FULLTEXT INDEX ...` where Rails emits
`CREATE fulltext INDEX ...`. Any test asserting on generated SQL text sees the
difference, and the two sides disagree about what `index.type` is allowed to
carry.

## Converged shape

Drop the `.toUpperCase()` so the arm is `if (index.type) parts.push(index.type)`,
matching `schema_creation.rb:114`.

Check before deleting: find who sets `index.type` on the trails side and
confirm the stored value is already the case Rails stores. If trails normalises
to lower-case at the `add_index` boundary the way Rails does, the uppercase is
pure invention and the deletion is complete. If some trails caller stores an
upper-case type, converge that caller instead — the fix belongs at whichever
end diverged from Rails, not split across both.

Expect SQL-text assertions to move. Per the MySQL-backticks note in the repo,
quoted identifiers differ per adapter, so assert on the type token rather than
whole-statement equality where a test needs updating.

## Acceptance criteria

- [ ] `visit_CreateIndexDefinition`'s type arm mirrors
      `schema_creation.rb:114` with no case transformation.
- [ ] `index.type`'s stored casing is confirmed to match Rails at the point it
      is set; if it did not, that call site is converged instead.
- [ ] MySQL/MariaDB `fulltext` and `spatial` index creation still works, with
      any SQL-text assertion updated to the Rails spelling rather than the test
      being loosened.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.
