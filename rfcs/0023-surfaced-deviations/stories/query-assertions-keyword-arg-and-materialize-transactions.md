---
title: "Query assertions: schema toggle should be a defaulted option, not a required positional"
status: draft
updated: 2026-07-30
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
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

## Context

Rails' query assertions take the schema toggle as a **defaulted keyword arg**
(`vendor/rails/activerecord/lib/active_record/testing/query_assertions.rb:18`):

```ruby
def assert_queries_count(count = nil, include_schema: false, &block)
  ActiveRecord::Base.lease_connection.materialize_transactions
  ...
```

so every Rails call site reads `assert_queries_count(1) { ... }`.

trails made it a **required positional middle parameter**
(`packages/activerecord/src/testing/query-assertions.ts:60-64`):

```ts
export async function assertQueriesCount(
  count: number | undefined,
  includeSchema = false,
  fn: () => void | Promise<void>,
): Promise<void>;
```

Because `fn` sits third, the default on `includeSchema` is unreachable: callers
must write `assertQueriesCount(1, false, fn)`. Passing the Rails-shaped
`assertQueriesCount(1, fn)` fails at runtime with `TypeError: block is not a
function` from `Notifications.subscribed` — a confusing failure that surfaced
while porting a test on #5623. `assertNoQueries(includeSchema, fn)`,
`assertQueriesMatch`, and `assertNoQueriesMatch` share the shape.

Separately, the port omits Rails' `materialize_transactions` call on line 19, so
a lazily-started transaction can be counted as a query inside the block rather
than being flushed before counting.

## Acceptance criteria

- The schema toggle moves out of the positional path — trailing options object
  (`assertQueriesCount(1, fn)` / `assertQueriesCount(1, fn, { includeSchema:
true })`) or equivalent — so Rails-shaped call sites work and the default is
  reachable. Apply to `assertQueriesCount`, `assertNoQueries`,
  `assertQueriesMatch`, `assertNoQueriesMatch`.
- `materializeTransactions` is invoked before counting, mirroring
  `query_assertions.rb:19`.
- Existing call sites updated; `pnpm parity:api --arity` shows no new arity
  mismatch for these four.

## Absorbed: `query-assertions-missing-materialize-transactions`

Merged in during the RFC 0023 triage pass (2026-08-18). Original title: "assert_queries_count omits lease_connection.materialize_transactions"

### Context

Rails' `assert_queries_count` opens with
`ActiveRecord::Base.lease_connection.materialize_transactions`
(`vendor/rails/activerecord/lib/active_record/testing/query_assertions.rb:19`),
and `assert_queries_match` does the same. trails'
`packages/activerecord/src/testing/query-assertions.ts` has no `leaseConnection`
call at all, so pending transactions are not materialized before the query
count starts — any lazily-materialized `BEGIN` lands inside the counted window.

Surfaced by PR #5520: a helper there was briefly named `leaseConnection` with an
argument, which flipped `isPortedWithArgs` (`scripts/api-compare/compare.ts:236`)
and made every `lease_connection` call in the package significant to the wide
calls gate. The helper was renamed, so the gate no longer reports it — but the
missing call is real and independent of that accident. Same omission exists in
`connection-handling.ts#connection`, `tasks/database-tasks.ts#migration_connection`,
and `activerecord-test-support`'s `adapter-helper.ts#current_adapter?` /
`load-schema-helper.ts#load_schema`; several of those are deliberate (trails'
`leaseConnection` is async, so sync callers use `leaseConnectionSync`), so each
needs its own verdict.

### Acceptance criteria

- `assert_queries_count` / `assert_queries_match` materialize transactions
  before counting, matching query_assertions.rb:19 (or the deviation is
  justified at the call site if trails' async lease makes it impossible).
- The other four `lease_connection` sites are triaged: implemented, or given a
  wide-exclude entry with a per-entry reason naming the sync-lease deviation.
- `pnpm parity:api:calls` stays green.
