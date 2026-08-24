---
title: "PG lookup_cast_type_from_column drops Rails' verify!-when-type_map-nil guard"
status: draft
updated: 2026-08-17
rfc: "0119-connection-adapter-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 150
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

# PG `lookup_cast_type_from_column` drops Rails' `verify!` guard

## Context

Rails, `activerecord/lib/active_record/connection_adapters/postgresql/
quoting.rb:189-192`:

```ruby
def lookup_cast_type_from_column(column) # :nodoc:
  verify! if type_map.nil?
  type_map.lookup(column.oid, column.fmod, column.sql_type)
end
```

PR #6663 converged this member — it deleted the adapter's duplicate body and
routed `PostgreSQLAdapter` through the module port, and it converged
`typeMap.fetch(..., () => new ValueType())` onto Rails' `type_map.lookup(...)`.
It did NOT converge the first line. The port at
`packages/activerecord/src/connection-adapters/postgresql/quoting.ts`
(`lookupCastTypeFromColumn`) reads `this.typeMap` directly and carries a JSDoc
saying the lazy getter stands in for the guard.

That justification is only partly true, which is why this is filed rather than
closed:

- trails' `typeMap` getter (`postgresql-adapter.ts`, around :948-958) lazily
  builds a `HashLookupTypeMap` and runs `initializeInstanceTypeMap`. So the
  `nil` case genuinely cannot be observed, and the lookup never NPEs.
- But Rails' `verify!` is not a map builder. It is
  `AbstractAdapter#verify!` (`abstract_adapter.rb:778-781`) — reconnect if the
  connection is dead, then `configure_connection`, which is what loads the
  OIDs. trails' getter does none of that. A caller reaching this method on a
  dropped connection gets a silently empty map and a `ValueType` for every
  column, where Rails would reconnect and answer the real type.

The blocker for a literal port is that trails' `verify!` is `verifyBang` and is
**async**, while this method has sync callers (the attribute-read type caster,
`type-caster/connection.ts:33`, and `model-schema.ts:1653`). Awaiting at this
site is not available, and a floating `void this.verifyBang()` would be worse
than the omission — it would return a stale answer while a reconnect raced.

## Converged shape

Pick one and do it properly; do not re-document the omission.

- Preferred: make the OID-load path reach a sync-safe verify. If the sync
  callers can be shown to always run behind an already-configured connection
  (`execQuery` calls `loadAdditionalTypes` first), prove it and encode the
  invariant as an assertion at the sync boundary rather than as prose.
- Otherwise: converge the callers to async so `verifyBang` can be awaited here,
  which is the same shape RFC 0073 is driving elsewhere.
- If neither is reachable, `pnpm tasks block` with the specific caller that
  cannot become async — not with "TS has no sync await".

## Acceptance criteria

- [ ] `lookupCastTypeFromColumn` either performs Rails' `verify!`-when-nil
      behaviour or the invariant that makes it unnecessary is asserted, not
      asserted-in-prose.
- [ ] The behaviour on a dropped connection is covered by a test: Rails
      reconnects and answers the real type.
- [ ] `pnpm parity:api:calls` green; no new baseline row for `verify!`.
