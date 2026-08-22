---
title: "Visit an ActiveModel::Attribute directly in the Arel ToSql visitor"
status: ready
updated: 2026-08-22
rfc: "0112-one-rails-thing-n-trails-things"
cluster: duplicate-bodies
packages: []
deps: []
deps-rfc: []
est-loc: 120
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

# Visit an ActiveModel::Attribute directly in the Arel ToSql visitor

## Context

Landed with PR #6429 (RFC 0084), which converged
`Persistence#_create_record`/`#_update_row` onto Rails'
`attributes_with_values` (`vendor/rails/activerecord/lib/active_record/attribute_methods.rb:503-505`,
`attribute_names.index_with { |name| @attributes[name] }`). The map now carries
`ActiveModel::Attribute` objects, as Rails' does.

Rails hands that map to Arel verbatim:

```ruby
im.insert(values.transform_keys { |name| arel_table[name] })   # persistence.rb:247
um.set(values.transform_keys { |name| arel_table[name] })      # persistence.rb:~330
```

and `Arel::Visitors::ToSql#visit_ActiveModel_Attribute` (`arel/visitors/to_sql.rb`)
binds each one via `collector.add_bind(o)`.

trails' visitor has no such dispatch: the only node that binds is
`Arel::Nodes::BindParam`, so `_insertRecord` and `_updateRecord`
(`packages/activerecord/src/persistence.ts:261-268`, `:327-333`) wrap each value
in `new Nodes.BindParam(val)` at the exact line Rails does the key transform.
The wrap is behaviour-neutral — `visitArelNodesBindParam`
(`packages/arel/src/visitors/to-sql.ts:1369`) and `SubstituteBinds`
(`packages/arel/src/collectors/substitute-binds.ts:9-13`) both unwrap
`valueForDatabase` off the wrapped Attribute — but it is an extra node Rails'
bodies do not build, in two ported method bodies.

## Converged shape

`Arel::Visitors::ToSql` dispatches an `ActiveModel::Attribute` value the way
Rails' `visit_ActiveModel_Attribute` does (bind it), so `_insertRecord` and
`_updateRecord` can pass the `attributes_with_values` map to `im.insert` /
`um.set` unwrapped, line-for-line with `persistence.rb`.

Note the dispatch has to cover the duck-typed detection trails already needs
elsewhere (`quoting.ts:552-562` documents why `instanceof ModelAttribute` alone
silently misses when two copies of `@blazetrails/activemodel` resolve).

## Acceptance criteria

- [ ] The ToSql visitor binds a bare `ActiveModel::Attribute` value.
- [ ] `_insertRecord`/`_updateRecord` build no `BindParam` of their own; the
      values reach Arel as Rails' do.
- [ ] Both the prepared (`compileWithBinds`) and the unprepared
      (`SubstituteBinds`) paths produce the same SQL and binds as today.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
