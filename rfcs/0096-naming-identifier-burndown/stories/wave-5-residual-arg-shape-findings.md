---
title: "wave-5-residual-arg-shape-findings"
status: claimed
updated: 2026-08-23
rfc: "0096-naming-identifier-burndown"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: null
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

## Acceptance criteria

1. Each row above is either converged (the TS body passes what Rails passes) or
   `pnpm tasks block`ed with the specific blocker. Renaming is not a valid
   close for any of them.
2. `pnpm parity:api:calls:args:report` shows these rows gone from the
   convergeable population; no new `call-mismatches-exclude/` row is added.
3. The `query-cache.ts#compute_if_absent` entry is treated as a behaviour bug —
   the LRU touch (`@map.delete(key)` then re-insert) and `@map.shift` eviction
   are ported, with a regression test that fails on the current baseline.
4. `pnpm build && pnpm test` green; `pnpm parity:api:calls` and
   `pnpm parity:api:calls:args` stay green.
