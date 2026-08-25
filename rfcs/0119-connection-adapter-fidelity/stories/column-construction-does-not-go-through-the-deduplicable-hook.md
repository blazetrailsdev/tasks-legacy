---
title: "column-construction-does-not-go-through-the-deduplicable-hook"
status: done
updated: 2026-08-25
rfc: "0119-connection-adapter-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: 7047
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced in PR #7047 review (RFC 0119,
`column-bypasses-deduplicable-registry`).

That PR ported `Column#deduplicateKey` / `deduplicate` / `deduplicated`
(`column.rb:75-112`, `deduplicable.rb:18-26`) and routed `deduplicate()` through
the shared registry. What it did NOT do is make construction transparent.

In Rails, `Deduplicable::ClassMethods#new` (`deduplicable.rb:13-14`) wraps
`Column.new` itself:

```ruby
def new(*, **)
  super.deduplicate
end
```

so there is no way to build a non-deduplicated Rails `Column`. trails'
`Column` constructor does not deduplicate, and the three production
construction sites — the `new_column_from_field` ports at
`connection-adapters/postgresql-adapter.ts` (`newColumnFromField`),
`connection-adapters/sqlite3/schema-statements.ts` (`newColumnFromField`) and
`connection-adapters/mysql/schema-statements.ts` (`newColumnFromField`) — all
`new Column(...)` directly. So real reflected columns are still distinct,
un-frozen objects.

## Why the obvious fix is blocked

Adding `.deduplicate()` at those three sites was tried in #7047 and REVERTED,
because it is not behaviour-preserving on two of the three adapters:

- `SQLite3::Column#==` / `#hash` (`sqlite3/column.rb:47-58`) fold in
  `auto_increment?` and `rowid` but NOT `@generated_type`, even though
  `virtual?` (`:24-26`) and `virtual_stored?` (`:28-30`) read it.
- `PostgreSQL::Column#==` / `#hash` (`postgresql/column.rb:64-77`) fold in
  `identity?` and `serial?` but NOT `@generated`, even though `virtual?`
  (`:27-30`) reads it.

So two columns identical in name, type, nullability and default but differing in
generated-ness are `eql?` in Rails and collapse to one registry entry. That is
reachable in production — the same column name in two different tables is
enough. With the wiring in place,
`sqlite3/schema-statements.trails.test.ts > SQLite3::SchemaStatements >
newColumnFromField > marks generated stored columns` fails: the stored column
collapses onto the virtual one registered by the preceding test and reports
`isVirtualStored() === false`.

This looks like a genuine upstream Rails hole rather than a port bug. It needs a
decision before the hook can be wired, and that decision is this story.

## Acceptance criteria

- [ ] Confirm the upstream behaviour against real Rails (build two SQLite
      columns differing only in `generated_type` and check `eql?` / registry
      identity) — `ruby` is on PATH; do not derive it from the source alone.
- [ ] Either mirror Rails exactly (accepting the collapse, with the SQLite/PG
      reflection consequences understood and any affected trails test
      re-derived from the Rails behaviour, NOT reworded), or carry the
      generated-state in the trails `deduplicateKey` with the divergence
      justified at the call site against `sqlite3/column.rb:53-58` and
      `postgresql/column.rb:72-77`.
- [ ] Once resolved, construction goes through the registry for all three
      adapters, as `deduplicable.rb:13-14` does.
- [ ] A test asserts two identically-reflected columns from a real adapter
      round-trip are the same instance, failing on baseline.
