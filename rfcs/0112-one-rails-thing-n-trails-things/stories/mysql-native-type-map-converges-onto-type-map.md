---
title: "Converge MySQL's _nativeTypeMap/lookup_cast_type onto the inherited type_map path"
status: ready
updated: 2026-08-22
rfc: "0112-one-rails-thing-n-trails-things"
cluster: split-stores
deps: []
deps-rfc: []
est-loc: null
priority: 40
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Rails has no MySQL override of `lookup_cast_type` or
`lookup_cast_type_from_column` — only `abstract/quoting.rb:234-236` and
`postgresql/quoting.rb:189-196`. trails carries a MySQL pair at
`packages/activerecord/src/connection-adapters/abstract-mysql-adapter.ts:1420-1430`,
backed by the invented `_nativeTypeMap` / `_buildTypeMap` slots.

PR #5520 ported `AbstractAdapter::TYPE_MAP` (abstract_adapter.rb:942),
`AbstractAdapter.extended_type_map` (877-883) and
`AbstractAdapter#lookup_cast_type` (abstract/quoting.rb:234), and reshaped
`AbstractMysqlAdapter.extended_type_map` onto Rails' `super` + `tinyint(1)`
form (abstract_mysql_adapter.rb:702-708). The inherited path therefore works
for MySQL now, and the override is a duplicate.

**Ordering hazard — do the declarations FIRST.** Deleting `_nativeTypeMap` is
what makes the missing per-class `TYPE_MAP` bite. Today
`AbstractAdapter.extended_type_map` seeds with `new TypeMap(this.TYPE_MAP)`
(abstract-adapter.ts, mirroring abstract_adapter.rb:878) and `this.TYPE_MAP`
always resolves to the single abstract map, because trails declares the
constant only on `AbstractAdapter`. In Ruby the same line resolves
`self::TYPE_MAP` to `Mysql2Adapter::TYPE_MAP` (mysql2_adapter.rb:53), which
carries MySQL's `initialize_type_map` registrations (`tinytext`, `tinyblob`,
`mediumint`, unsigned variants). That divergence is inert only while
`_nativeTypeMap` is the real casting path — the moment this story deletes it,
MySQL starts casting through the abstract map and silently resolves the wrong
types for every MySQL-specific sql_type. The same applies to `SQLite3Adapter`
(sqlite3_adapter.rb:505-506), which also lacks both constants and currently
shares `AbstractAdapter.EXTENDED_TYPE_MAPS`.

Two Rails details the convergence depends on:

- Rails declares `TYPE_MAP` on `Mysql2Adapter` (mysql2_adapter.rb:53) and
  `SQLite3Adapter` (sqlite3_adapter.rb:505), seeded from each class's own
  `initialize_type_map`. trails declares it only on `AbstractAdapter`, so
  `self::TYPE_MAP` inside `extended_type_map` resolves to the abstract map for
  MySQL rather than MySQL's own — the MySQL-specific registrations
  (`tinytext`, `mediumint`, … abstract_mysql_adapter.rb:711+) live only in
  `_buildTypeMap` and are not ported to `initialize_type_map` at all.
- `emulate_booleans` is part of MySQL's `extended_type_map_key`
  (abstract_mysql_adapter.rb:762-768), which trails already mirrors.

## Acceptance criteria

- `AbstractMysqlAdapter` declares its MySQL `initialize_type_map` overrides,
  and `Mysql2Adapter` (plus `SQLite3Adapter`, sqlite3_adapter.rb:505-506)
  declare their own `TYPE_MAP` / `EXTENDED_TYPE_MAPS`, matching Rails.
- `_nativeTypeMap` / `_buildTypeMap` and the MySQL `lookup_cast_type` /
  `lookup_cast_type_from_column` overrides are deleted; casting resolves
  through the inherited `type_map` → `extended_type_map` path.
- `default_timezone` reaches MySQL datetime casting (this subsumes
  `mysql-extended-type-map-default-timezone`).
- MySQL lane green (`ARCONN=mysql2`); SQLite lane unaffected.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
