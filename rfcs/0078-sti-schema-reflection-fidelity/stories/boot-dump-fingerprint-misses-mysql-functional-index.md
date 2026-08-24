---
title: "Boot-dump fingerprint misses a MySQL functional index's expression"
status: in-progress
updated: 2026-08-24
rfc: "0078-sti-schema-reflection-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 80
priority: null
pr: 6977
claim: "2026-08-24T11:45:44Z"
assignee: "boot-dump-fingerprint-misses-mysql-functional-index"
blocked-by: null
closed-reason: null
---

## Context

The boot-dump replay guard added in PR #6972 hashes each lane's index metadata
so a canonical table whose indexes changed cannot be answered from the dump
(`packages/activerecord/src/support/schema-cache-dump.ts`, `SHAPE_QUERIES`).
The MySQL arm reads `information_schema.STATISTICS` and deliberately omits its
`EXPRESSION` column:

- MySQL 8 exposes a functional index's expression there, with `COLUMN_NAME`
  NULL — which is what Rails' MySQL adapter reads off `SHOW KEYS`' `Expression`
  to build `IndexDefinition#columns`
  (`activerecord/lib/active_record/connection_adapters/mysql/schema_statements.rb:36-52`).
- MariaDB's `STATISTICS` has no `EXPRESSION` column at all; selecting it fails
  with `ER_BAD_FIELD_ERROR` (verified against `mariadb:11`).

So a functional index on a canonical table whose _expression_ changed, with its
column set unchanged, would leave the MySQL fingerprint intact and could replay
a stale `IndexDefinition`. Harmless today only because the canonical schema has
no functional index — an assumption nothing enforces.

## Acceptance criteria

- The MySQL index fingerprint covers a functional index's expression on servers
  that have it, without breaking MariaDB — a server-version branch (the lane
  already has `supportsExpressionIndex?`-style capability predicates to key off,
  see `abstract-mysql-adapter.ts`) or reading `SHOW KEYS` instead of
  `information_schema`, which is what Rails reads and which carries
  `Expression` on both servers where it exists.
- A test that changes a functional index's expression on a canonical table and
  asserts the fingerprint moves, skipped where the server lacks the feature.
- The `EXPRESSION is left out` note in `SHAPE_QUERIES`' JSDoc goes away with it.
