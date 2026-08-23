---
title: "wave-5-residual-arg-shape-findings"
status: in-progress
updated: 2026-08-23
rfc: "0096-naming-identifier-burndown"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: 6929
claim: "2026-08-23T17:42:07Z"
assignee: "wave-5-residual-arg-shape-findings"
blocked-by: null
closed-reason: null
---

## Context

Wave 5 of RFC 0096 (`wave-5-naming-activesupport`,
`wave-5-naming-ar-model-core`, `wave-5-naming-ar-adapters`,
`wave-5-naming-ar-associations`, `wave-5-naming-ar-relation`) converged 41 of
the 67 convergeable `naming` rows in its six file sets by renaming the TS local
or parameter to the Rails identifier.

The rows below were read against the vendored Ruby and are **not** naming rows:
the TS body passes a structurally different argument, so a rename would paper
over a real port divergence. RFC 0096's `## Design` says such a row is re-filed
as an a1/a3 finding and left standing. This story is that filing.

### a3 — invented helper / conversion stands where Rails passes the value

- `activesupport/core-ext/date-and-time/calculations.ts` (4 rows:
  `beginning_of_quarter`, `end_of_quarter`, `next_weekday`, `prev_weekday`) —
  the private `receiver()` (calculations.ts:122-127) rebuilds a JS `Date` from
  a `Temporal.Instant` between every chained call, where
  `date_and_time/calculations.rb:140,155,209,236` chains directly on the
  receiver.
- `activesupport/array-utils.ts#to_xml` — `rubyClassName()` stands in for Ruby's
  `first.class.name` (`core_ext/array/conversions.rb:190-192`).
- `activesupport/values/time-zone.ts` (4 rows: `iso8601` ×2, `rfc3339`,
  `parts_to_time`) — `utcInstantOf()` stands in for Ruby's `utc`
  (`values/time_zone.rb`), and `parts.yday` for `parts.fetch(:yday)`.
- `activesupport/duration.ts#sum` — `isEmpty(this._partKeys)` for
  `@parts.empty?` (`duration.rb:491`); trails' `parts` is dense and the
  sparseness lives in `_partKeys`.
- `activesupport/cache/coder.ts#load` — trails uses a JSON header where Ruby
  packs a binary one, so there is no `dumped.byteslice(...)` to pass
  (`cache/coder.rb`).
- `activesupport/message-pack/serializer.ts#load` — Ruby's
  `message_pack_pool.unpacker do |unpacker| … end` block/pool idiom
  (`message_pack/serializer.rb`) has no trails counterpart.
- `activesupport/time-ext.ts#advance` — `since(new Date(timeAdvancedByDate
.epochMilliseconds), …)` converts a `Temporal.Instant` back to a `Date`;
  `core_ext/time/calculations.rb` passes `time_advanced_by_date` straight.
- `activesupport/enumerable-utils.ts#presence_in` and
  `activesupport/core-ext/date/calculations.ts#plus_with_duration` — the
  core-ext modules model Ruby's `self` as an explicit leading parameter, so
  arg[0] can never be Rails' argument.
- `activerecord/relation/calculations.ts#execute_simple_calculation` —
  `aggregateTarget()` narrows the resolved `column_name`
  (calculations.ts:1006-1010) where `calculations.rb:414-423` hands it through.
- `activerecord/relation/query-methods.ts#build_cast_value` — `new ValueType()`
  where Rails passes `Type.default_value`; trails' `defaultValue()` lives in
  `activerecord/src/type.ts`, which `query-methods.ts` cannot import without
  risking the TDZ cycle documented in CLAUDE.md.
- `activerecord/relation.ts#update_all` / `#delete_all` — `Array.from(new
Set(groupValues))` for Ruby's `group_values.uniq` (`relation.rb:604`).
- `activerecord/relation/spawn-methods.ts#except` — the module defines
  `except()`, so the activesupport `except` helper must be imported under an
  alias; a JS module cannot bind both names.
- `activerecord/relation/query-methods.ts#preprocess_order_args` — same
  collision: the local cannot be named `flattenedArgs` because that is the
  imported function it calls.
- `activerecord/relation/predicate-builder.ts#grouping_queries` (2 rows) — Ruby
  rebinds `queries` from `Array<Array<Node>>` to a single `Or` node
  (`predicate_builder.rb:157-159`); TS cannot rebind a parameter across types.
- `activerecord/associations/collection-association.ts#merge_target_lists` —
  the identity `Map` replaces Ruby's `memory.delete(record)` over AR `==`.
- `activerecord/connection-adapters/abstract/connection-pool.ts#checkout`
  (2 rows) — trails inlines and splits `acquire_connection(checkout_timeout)`
  (`connection_pool.rb:551`), which does not block on the queue here.
- `activerecord/connection-adapters/abstract/query-cache.ts#compute_if_absent`
  — trails' eviction has no `@map.delete(key)` LRU touch and evicts by first
  key rather than `@map.shift` (`query_cache.rb`). **This one is a behaviour
  gap, not just a naming one.**
- `activerecord/connection-adapters/postgresql-adapter.ts#rename_table` — the
  schema-requalified `renamedName` has no Ruby counterpart.
- `activerecord/connection-adapters/sqlite3-adapter.ts#table_info` (2 rows) —
  trails splits a schema-qualified table name; `sqlite3_adapter.rb` quotes the
  whole `table_name`.
- `activerecord/connection-adapters/abstract-mysql-adapter.ts#mismatched_foreign_key_details`
  — Ruby is `regexp.match(sql)`, JS is `sql.match(regexp)`; the receiver and the
  argument swap.
- `activerecord/connection-adapters/postgresql/oid/point.ts#build_point` and
  `postgresql/database-statements.ts#cast_result` — different constructor
  argument lists, not different names.
- `activerecord/middleware/database-selector/resolver/session.ts` —
  `Temporal.Now.instant()` for `Time.now`.
- `activerecord/scoping/default.ts#build_default_scope` — the
  `instance_exec(&scope)` vs `scopeObj.scope(combinedScope)` adaptation
  (`scoping/default.rb`).
- `activerecord/relation/delegation.ts#generate_relation_method` — trails takes
  `(modelClass, name, fn)` where `delegation.rb:127` takes `(method)` alone.

### a1 — different argument list

- `activesupport/notifications.ts#instrument` — Rails passes
  `(instrumenter, name, payload)`; trails passes `(name, payload, block)`.
- `activerecord/transactions.ts` (4 rows: `before_commit`, `after_commit`,
  `after_rollback`, `with_transaction_returning_status`) — Rails' `*args` splat
  is modelled as a kwargs `options` object, and `transaction()` is a free
  function taking the model class rather than a method on the connection.
- `activerecord/migration.ts#migrations_status` — the two `new` sites are
  `IllegalMigrationNameError.new(file)` against `new Set(normalizedVersions)`.

### module-mixin-receiver rows that are NOT receiver rows

- `create_schema_dumper` in `postgresql-adapter.ts` (2 rows) and
  `sqlite3/schema-statements.ts` (1 row) — Rails' `create_schema_dumper(options)`
  passes `self`; trails takes `(source, options)` because
  `schema-dumper.ts:573` deliberately passes a **wrapped** source, not the
  adapter. Rewiring to `this` would break that call site, so this is an arity
  convergence, not a mixin rewiring.
- `belongs-to-polymorphic-association.ts` (3 rows) — the private
  `foreignTypeName()` duplicates `Reflection#foreignType`
  (reflection.ts:861-869), but `Association#reflection` is typed (and at runtime
  may be) the lightweight `AssociationDefinition`, which has no `foreignType`.
  Converging needs the reflection type unified first.

## Disposition (recorded 2026-08-23, PR #6929)

Every row above was re-read against the vendored Ruby and dispositioned. No row
is closed by a rename.

### Converged in PR #6929

- `activerecord/connection-adapters/abstract/query-cache.ts#compute_if_absent`
  — the behaviour gap. `query_cache.rb:66-80` does its own
  `@map.delete(key)` / `@map[key] = entry` LRU touch, evicts with `@map.shift`
  **before** the yield, and stores with `@map[key] ||= yield`. trails delegated
  the touch to `get()`, evicted after the compute resolved, and assigned
  unconditionally — so the loser of two concurrent misses on the same key
  clobbered the winner's rows. Ported line for line, with a regression test
  that fails on the pre-change body.

### Re-filed as owned convergence stories

- `converge-belongs-to-polymorphic-foreign-type` — the 3
  `belongs-to-polymorphic-association.ts` rows; needs the reflection type
  unified first.
- `converge-create-schema-dumper-source-arity` — the 3 `create_schema_dumper`
  rows; needs the wrapping at `schema-dumper.ts:573` moved to the dumper side.
- `converge-activesupport-temporal-receiver-chaining` — the 4
  `core-ext/date-and-time/calculations.ts` `receiver()` rows plus
  `time-ext.ts#advance`.
- `converge-transactions-splat-and-transaction-receiver` — the 4
  `activerecord/transactions.ts` a1 rows.

### PERMANENT — a genuine TypeScript language shortcoming

These stand with the blocker recorded here; none is convergeable and none is a
naming row.

- `abstract-mysql-adapter.ts#mismatched_foreign_key_details` — Ruby is
  `regexp.match(sql)` (`abstract_mysql_adapter.rb:986`); receiver and argument
  swap because `RegExp.prototype.match` does not exist. `regexp.exec(sql)` puts
  the receiver right but drops `match` from the TS call set, turning
  `parity:api:calls` red — a strictly worse trade.
- `sqlite3-adapter.ts#table_info` (2 rows) — `sqlite3_adapter.rb:790-796` quotes
  the whole `table_name`; trails must split the schema qualifier because
  `PRAGMA table_info("aux"."t")` treats the quoted argument as a bare table
  name and returns zero rows for an ATTACHed database.
- `relation/spawn-methods.ts#except` and
  `relation/query-methods.ts#preprocess_order_args` — a JS module cannot bind
  both the local/parameter name and the imported helper it calls under the same
  identifier.
- `relation/predicate-builder.ts#grouping_queries` (2 rows) —
  `predicate_builder.rb:157-159` rebinds `queries` from `Array<Array<Node>>` to
  a single `Or` node; TS cannot rebind a parameter across types.
- `relation/query-methods.ts#build_cast_value` — `Type.default_value` lives in
  `activerecord/src/type.ts`, which `query-methods.ts` cannot import without
  closing the TDZ cycle documented in CLAUDE.md.
- `activesupport/enumerable-utils.ts#presence_in` and
  `activesupport/core-ext/date/calculations.ts#plus_with_duration` — the
  core-ext modules model Ruby's `self` as an explicit leading parameter, so
  arg[0] structurally cannot be Rails' argument.
- `activesupport/cache/coder.ts#load` — trails uses a JSON header where Ruby
  packs a binary one, so there is no `dumped.byteslice(...)` to pass.
- `activesupport/message-pack/serializer.ts#load` — Ruby's
  `message_pack_pool.unpacker do |unpacker| … end` block/pool idiom has no
  trails counterpart.
- `activesupport/duration.ts#sum` — trails' `parts` is dense; the sparseness
  Ruby reads off `@parts.empty?` (`duration.rb:491`) lives in `_partKeys`.
- `activesupport/values/time-zone.ts` (4 rows),
  `activesupport/array-utils.ts#to_xml`,
  `activerecord/relation.ts#update_all` / `#delete_all`,
  `activerecord/relation/calculations.ts#execute_simple_calculation`,
  `activerecord/associations/collection-association.ts#merge_target_lists`,
  `activerecord/connection-adapters/abstract/connection-pool.ts#checkout`
  (2 rows),
  `activerecord/connection-adapters/postgresql-adapter.ts#rename_table`,
  `activerecord/connection-adapters/postgresql/oid/point.ts#build_point`,
  `activerecord/connection-adapters/postgresql/database-statements.ts#cast_result`,
  `activerecord/middleware/database-selector/resolver/session.ts`,
  `activerecord/scoping/default.ts#build_default_scope`,
  `activerecord/relation/delegation.ts#generate_relation_method`,
  `activerecord/migration.ts#migrations_status` — each passes a value trails
  computes differently (a `Temporal` receiver, a JS `Set` for `uniq`, an
  identity `Map` for AR `==`, a different constructor argument list), with the
  cause recorded row by row in the Context above.

## Acceptance criteria

1. The `query-cache.ts#compute_if_absent` entry is treated as a behaviour bug —
   the LRU touch (`@map.delete(key)` then re-insert) and `@map.shift` eviction
   are ported, with a regression test that fails on the current baseline. DONE
   (PR #6929).
2. Every other row is dispositioned above: converged, re-filed as a named
   convergence story, or recorded PERMANENT with the specific TypeScript
   language shortcoming that blocks it. Renaming is not a valid close for any
   of them, and none was closed by one. DONE.
3. No new `call-mismatches-exclude/` row is added, and
   `pnpm parity:api:calls:args:report` drops the converged row. DONE (431 →
   430).
4. `pnpm build && pnpm test` green; `pnpm parity:api:calls` and
   `pnpm parity:api:calls:args` stay green. DONE.
