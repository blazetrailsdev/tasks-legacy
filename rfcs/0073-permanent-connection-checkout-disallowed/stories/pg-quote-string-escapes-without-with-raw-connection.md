---
title: "PostgreSQL quote_string escapes without taking with_raw_connection's lease"
status: draft
updated: 2026-08-24
rfc: "0073-permanent-connection-checkout-disallowed"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 80
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`PostgreSQL::Quoting#quote_string` is
`with_raw_connection { |connection| connection.escape(s) }`
(`activerecord/lib/active_record/connection_adapters/postgresql/quoting.rb`),
so the escape happens inside a lease on the raw libpq handle.

`packages/activerecord/src/connection-adapters/postgresql/quoting.ts`
`quoteString` escapes with the pure PG rules instead and takes no lease,
because `withRawConnection` is async in trails while `quoteString` is reached
from synchronous quoting paths (`quote`, and every caller below it). PR #7008
replaced its call-set baseline row with a `@missingRailsCall with_raw_connection
— CONVERGEABLE` receipt at the call site; this story retires the receipt.

The escaping itself already matches Rails (PG `standard_conforming_strings`:
double `'`, backslash is ordinary), so the divergence is the missing lease, not
the output.

## Converged shape

`quoteString` takes the lease Rails takes — which requires the synchronous
quoting callers above it to become awaitable, the same sync/async lease flip the
rest of RFC 0073 is doing. Delete the `@missingRailsCall` tag in
`packages/activerecord/src/connection-adapters/postgresql/quoting.ts` when it
lands.

## Acceptance criteria

- [ ] `quoteString` escapes through `withRawConnection`, matching
      `postgresql/quoting.rb`.
- [ ] The `@missingRailsCall with_raw_connection` receipt is deleted, not
      reworded.
- [ ] `pnpm parity:api:calls` / `:args` green; SQLite, PostgreSQL and
      MySQL/MariaDB lanes green.
