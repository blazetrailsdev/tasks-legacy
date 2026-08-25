---
title: "Converge PG configure_connection back onto internal_execute/quote (non-re-entrant checkout)"
status: draft
updated: 2026-08-18
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 220
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`PostgreSQLAdapter#configure_connection`
(`vendor/rails/activerecord/lib/active_record/connection_adapters/postgresql_adapter.rb:956-997`)
issues its session setup through `internal_execute(..., "SCHEMA")` and renders
values with `quote(v)`:

```ruby
variables = @config.fetch(:variables, {}).stringify_keys      # :977
internal_execute("SET intervalstyle = iso_8601", "SCHEMA")    # :980
internal_execute("SET SESSION #{k} TO DEFAULT", "SCHEMA")     # :987
internal_execute("SET SESSION #{k} TO #{quote(v)}", "SCHEMA") # :989
```

trails displaces the whole body into the private
`_maybeConfigureConnection(client)` (`postgresql-adapter.ts:836-887`), which
writes the SET statements straight to the `pg.Client` and renders values with
`quoteLiteral` instead. `configureConnection()` still exists at the Rails name
(`postgresql-adapter.ts:2907`) but only delegates.

**Why it is currently displaced.** Rails' `with_raw_connection` wraps its body
in `@lock.synchronize` (`abstract_adapter.rb:983-984`), and `@lock` is a
`Monitor` (`abstract_adapter.rb:180-189`, assigning
`ActiveSupport::Concurrency::LoadInterlockAwareMonitor` /
`ThreadLoadInterlockAwareMonitor`, both `< Monitor` — see
`activesupport/lib/active_support/concurrency/load_interlock_aware_monitor.rb:32`).
`Monitor#synchronize` is **re-entrant for the owning thread**, so Rails can nest
`internal_execute` inside `configure_connection` while the acquire path already
holds the connection. trails' checkout is not re-entrant, so the nested call
deadlocks instead of re-entering — routing through `internalExecute` re-enters
`connectBang`/`verify`.

This was re-confirmed during RFC 0106 wave 4b (PR #6724) and the root cause is
now recorded on the four affected baseline rows in
`scripts/api-compare/call-mismatches-exclude/activerecord/connection-adapters/postgresql-adapter.json`.
That is a burndown ledger row, not permission — hence this story.

Secondary, same method, different root cause: the two `fetch` rows.
`@config.fetch(:variables, {})` (`postgresql_adapter.rb:977` and, for
`reconfigure_connection_timezone`, `:1000`) is re-read per call; trails reads the
frozen `_sessionVariables` parsed once at construction
(`postgresql-adapter.ts:734`).

## Converged shape

- Make the connection checkout re-entrant for the holder — a explicit
  re-entrancy token threaded through the acquire, since JS has no thread/fiber
  identity a promise-mutex can key on — so `configure_connection` can call
  `internalExecute` the way Rails does.
- Move the body back onto `configureConnection()` at the Rails name, argless and
  operating on the published raw connection, matching
  `postgresql_adapter.rb:956`.
- Render session-variable values with `quote(v)` (`:989`) rather than
  `quoteLiteral`, once the type-cast stack is safe to re-enter mid-configure.
- Read `@config[:variables]` at call time rather than the construction-time
  frozen copy.

## Acceptance criteria

- [ ] `configureConnection()` carries the body; `_maybeConfigureConnection` is gone
      or reduced to the once-per-socket gate.
- [ ] The SET statements go through `internalExecute(..., "SCHEMA")`.
- [ ] Values render through `quote()`.
- [ ] These rows are deleted from
      `call-mismatches-exclude/activerecord/connection-adapters/postgresql-adapter.json`:
      `configure_connection`/`internal_execute`, `configure_connection`/`quote`,
      `configure_connection`/`fetch`, `reconfigure_connection_timezone`/`raw_execute`,
      `reconfigure_connection_timezone`/`fetch` — by hand via `serializeBaseline`,
      then `pnpm parity:api:calls:tighten`. No `--write`, no reseed.
- [ ] `pnpm parity:api:calls` and `pnpm parity:api:calls:args` green.
- [ ] PostgreSQL lane green — SQLite is not evidence for any of this.

## Notes

Overlaps the checkout machinery, so it may need to land after or alongside
RFC 0073's pool-checkout flip. Sibling story
`pg-get-database-version-uses-with-raw-connection` shares the identical root
cause and could be bundled.
