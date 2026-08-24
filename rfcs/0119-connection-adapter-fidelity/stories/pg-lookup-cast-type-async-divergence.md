---
title: "Converge PostgreSQLAdapter#lookup_cast_type onto the sync base signature"
status: draft
updated: 2026-07-28
rfc: "0119-connection-adapter-fidelity"
cluster: null
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`PostgreSQLAdapter#lookupCastType`
(`packages/activerecord/src/connection-adapters/postgresql-adapter.ts:3633`) is
`async` — it issues `SELECT '<sql_type>'::regtype::oid` before looking the OID
up in the type map. Rails' is synchronous
(`vendor/rails/activerecord/lib/active_record/connection_adapters/postgresql/quoting.rb:195`),
as is the `AbstractAdapter#lookup_cast_type` ported in PR #5520
(`abstract/quoting.rb:234`), so PG's override diverges from the base signature:
it returns `Promise<Type>` where the base returns a `Type`.

PR #5520 also had to drop `private` from the override, since TS cannot narrow an
inherited member's visibility — Rails keeps it private.

Today the divergence is contained: PG overrides `lookupCastTypeFromColumn`
(sync, OID-keyed, `postgresql-adapter.ts:961`) and `quoteDefaultExpression`
(async, awaits the promise), so the sync duck-typed consumers
(`abstract-adapter.ts` `quoteDefaultExpression`, `abstract/database-statements.ts`
fixture serialization) never receive an unawaited promise. It is still a
latent trap: any new caller reaching PG through the base signature gets a
Promise and silently skips `serialize`.

## Acceptance criteria

- PG's `lookup_cast_type` either resolves synchronously (e.g. off an
  already-loaded OID/typname map, no `schemaQuery` round trip) or the
  divergence is recorded where the base method is declared, with the
  duck-typed call sites made promise-safe.
- Base and override signatures agree, so `lookupCastType` is honestly typed
  rather than `unknown`.
- PG lane green (`ARCONN=postgresql`).
