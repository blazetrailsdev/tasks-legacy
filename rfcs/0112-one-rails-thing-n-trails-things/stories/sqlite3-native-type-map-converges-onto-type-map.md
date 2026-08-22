---
title: "Converge SQLite3's _nativeTypeMap/lookup_cast_type onto the inherited type_map path"
status: ready
updated: 2026-08-22
rfc: "0112-one-rails-thing-n-trails-things"
cluster: duplicate-bodies
deps: []
deps-rfc: []
est-loc: 120
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`AbstractSQLite3Adapter#lookupCastType`
(`packages/activerecord/src/connection-adapters/sqlite3-adapter.ts`) has no
Rails counterpart: `vendor/rails/activerecord/lib/active_record/connection_adapters/sqlite3_adapter.rb`
only _calls_ `lookup_cast_type_from_column` (:628), and there is no
`sqlite3/quoting.rb` defining `lookup_cast_type` — the only definitions in Rails
are `abstract/quoting.rb:234-236` and `postgresql/quoting.rb:195`.

The override, plus the `_nativeTypeMap` / `_buildTypeMap` slots behind it and
its fetch-full-then-lookup-normalized fallback, is a trails invention — the same
shape as MySQL's, which `mysql-native-type-map-converges-onto-type-map` covers.
PR #5520 ported `AbstractAdapter::TYPE_MAP` (abstract_adapter.rb:942),
`AbstractAdapter.extended_type_map` (877-883) and the abstract
`lookup_cast_type`, so the inherited path exists; the override was left alone
there and its false `Mirrors:` line replaced with `@noRailsEquivalent`.

Ordering hazard, identical to the MySQL story: Rails declares `TYPE_MAP` and
`EXTENDED_TYPE_MAPS` on `SQLite3Adapter` (sqlite3_adapter.rb:505-506) and trails
declares neither, so `self::TYPE_MAP` inside `extended_type_map` resolves to the
abstract map. Deleting `_nativeTypeMap` before adding those declarations would
silently cast every SQLite-specific sql_type through the abstract map.

## Acceptance criteria

- `AbstractSQLite3Adapter` declares its SQLite `initialize_type_map` overrides,
  and `SQLite3Adapter` declares `TYPE_MAP` / `EXTENDED_TYPE_MAPS`
  (sqlite3_adapter.rb:505-506).
- `_nativeTypeMap` / `_buildTypeMap` and the `lookupCastType` override are
  deleted; casting resolves through the inherited `type_map` →
  `extended_type_map` path, including the precision/scale-bearing regex
  registrations the current fallback exists to serve.
- `lookup_cast_type_from_column` behaviour at sqlite3_adapter.rb:628 is
  unchanged.
- SQLite lane green (`ARCONN=sqlite3` and `sqlite3_mem`).

## Absorbed: `sqlite3-initialize-type-map-keeps-unaudited-registrations`

Merged in during the RFC 0023 triage pass (2026-08-18). Original title: "SQLite initializeTypeMap keeps unaudited registrations after super"

### Context

Rails' `SQLite3Adapter#initialize_type_map`
(`vendor/rails/activerecord/lib/active_record/connection_adapters/sqlite3_adapter.rb:499-502`)
is exactly two lines: `super`, then
`register_class_with_limit m, %r(int)i, SQLite3Integer`. Everything else the
SQLite map resolves comes from `AbstractAdapter#initialize_type_map`
(`abstract_adapter.rb:885-916`).

PR #5541 made `AbstractSQLite3Adapter.initializeTypeMap`
(`packages/activerecord/src/connection-adapters/sqlite3-adapter.ts:3040`) call
`super.initializeTypeMap(m)` so the base registrations and aliases are inherited
rather than hand-copied, and deleted the three aliases super already provides
(`clob`, `blob`, `number`). What remains after `super` is still far more than
Rails' single `%r(int)i` override: exact-string registrations for `string`,
`text`, `integer`, `float`, `decimal`, `boolean`, `blob`, `binary`, `json`,
`numeric`, `bigint`; a `/decimal|numeric/i` block that re-implements the base
map's decimal precision/scale extraction with its own inline regexes instead of
`extractScale`/`extractPrecision`; re-declared `/char/i`, `/text/i`, `/binary/i`
via `registerClassWithLimit`; a `/real|floa|doub/i` affinity registration; and
date/time/datetime re-registered so `datetime` lands on `SQLiteDateTimeType`.

Some of these are load-bearing and must survive: `SQLiteDateTimeType` exists
because the driver returns datetime columns as TEXT, the `bigint` entry and
`SQLite3IntegerType#_limit` split are enshrined by
`type_lookup_test.rb:84`'s `_limit` assertion, and the `/timestamp/i` alias must
stay registered _after_ the re-declared `/time/i` or reverse-registration lookup
resolves `timestamp` to Time instead of DateTime. The rest were never audited
against the base map — they are candidates for deletion, not fidelity.

### Acceptance criteria

- [ ] Enumerate every registration remaining in
      `AbstractSQLite3Adapter.initializeTypeMap` after the `super` call and
      classify each as (a) redundant with the inherited base entry — delete, or
      (b) a genuine SQLite override — keep with a justification at the call site
      naming the trails/driver reason.
- [ ] Replace the inline `/decimal|numeric/i` precision/scale regexes with the
      base map's `extractScale`/`extractPrecision` path (or delete the
      registration if the inherited `/decimal/i` block already covers it).
- [ ] Preserve the `/timestamp/i`-after-`/time/i` ordering invariant and the
      `bigint` / `_limit` behaviour; `connection-adapters/type-lookup.test.ts`
      (the port of `type_lookup_test.rb`) must stay green on both the sqlite3
      and sqlite3_mem lanes.
- [ ] No test renames and no invented assertions — the Rails assertion lists are
      already ported in full.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
