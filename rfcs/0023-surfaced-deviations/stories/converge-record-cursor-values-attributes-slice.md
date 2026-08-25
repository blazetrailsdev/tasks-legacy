---
title: "converge record_cursor_values onto attributes.slice"
status: draft
updated: 2026-08-17
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
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

`record_cursor_values`
(`vendor/rails/activerecord/lib/active_record/relation/batches.rb:408-410`) is:

```ruby
def record_cursor_values(record, cursor)
  record.attributes.slice(*cursor).values
end
```

trails (`packages/activerecord/src/relation/batches.ts`, `recordCursorValues`)
reads each column instead:

```ts
return cursor.map((column) => record.readAttribute?.(column) ?? record[column]);
```

Two divergences, the second load-bearing:

- Rails goes through the `attributes` hash once and slices it; the port calls
  `read_attribute` per column, and optional-chains it away entirely when the
  record has no such method.
- `?? record[column]` treats a **NULL column value as "missing"** and silently
  falls back to a plain JS property read, which can hit an association getter,
  a defined method, or `undefined`. Rails' `slice(*cursor).values` yields `nil`
  for a NULL attribute and never consults anything else. A cursor column that
  is NULL on some rows therefore sorts and compares against a different value
  here than in Rails (`compare_values_for_order` at batches.rb:414-424 receives
  it directly).

Surfaced while converging the `Kernel#Array` call sites in this file (PR #6633,
RFC 0096); the body itself was outside that story's scope.

## Acceptance criteria

- [ ] `recordCursorValues` mirrors `record.attributes.slice(*cursor).values` —
      one pass over the record's attributes, no per-column fallback chain.
- [ ] A NULL cursor-column value reaches `compareValuesForOrder` as null, not
      as whatever `record[column]` resolves to; covered by a test over a
      loaded relation batched on a nullable cursor column.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.
