---
title: "sweep-trails-only-test-files-connection-adapters"
status: ready
updated: 2026-08-23
rfc: "0078-sti-schema-reflection-fidelity"
cluster: null
packages: []
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

Follow-up slice of RFC 0078 `sweep-trails-only-test-files-onto-trails-name`,
whose first slice (`support/**`, 27 files) landed in trails#6932. That PR
regenerated the candidate list with the story's method — extract the TS test
manifest via `pnpm parity:test --package activerecord`, then take every
`packages/activerecord/src/**/*.test.ts` in
`scripts/test-compare/output/ts-tests.json` that never appears as a matched TS
counterpart in the per-file report. Result: **238** non-`.trails.` candidates,
of which **68** sit under `connection-adapters`:

```text
connection-adapters/abandon-raw-socket.test.ts
connection-adapters/abstract-adapter.lifecycle.test.ts
connection-adapters/abstract-adapter.query-logging.test.ts
connection-adapters/abstract-adapter.test.ts
connection-adapters/abstract-mysql-adapter.test.ts
connection-adapters/abstract/connection-pool/queue.test.ts
connection-adapters/abstract/database-limits.test.ts
connection-adapters/abstract/database-statements.test.ts
connection-adapters/abstract/precision-roundtrip.test.ts
connection-adapters/abstract/savepoints.test.ts
connection-adapters/abstract/schema-creation.test.ts
connection-adapters/abstract/schema-definitions.test.ts
connection-adapters/abstract/schema-dumper.test.ts
connection-adapters/abstract/schema-statements-on-adapter.test.ts
connection-adapters/abstract/schema-statements-privates.test.ts
connection-adapters/abstract/temporal-wire.test.ts
connection-adapters/adapter-args.test.ts
connection-adapters/column.test.ts
connection-adapters/dbconsole-option-keys.test.ts
connection-adapters/mysql/column.test.ts
connection-adapters/mysql/datetime-bind.test.ts
connection-adapters/mysql/explain-pretty-printer.test.ts
connection-adapters/mysql/quoting.test.ts
connection-adapters/mysql/schema-creation.test.ts
connection-adapters/mysql/schema-dumper.test.ts
connection-adapters/mysql/schema-statements.test.ts
connection-adapters/mysql/temporal-type-cast.test.ts
connection-adapters/pool-config.test.ts
connection-adapters/pool-manager.test.ts
connection-adapters/postgresql-adapter.check-version.test.ts
connection-adapters/postgresql-adapter.conn-params.test.ts
connection-adapters/postgresql-adapter.exec-query.test.ts
connection-adapters/postgresql-adapter.get-client.test.ts
connection-adapters/postgresql-adapter.type-map.test.ts
connection-adapters/postgresql/database-statements.test.ts
connection-adapters/postgresql/oid/array.test.ts
connection-adapters/postgresql/oid/cidr.test.ts
connection-adapters/postgresql/oid/date-time.test.ts
connection-adapters/postgresql/oid/date.test.ts
connection-adapters/postgresql/oid/hstore.test.ts
connection-adapters/postgresql/oid/jsonb.test.ts
connection-adapters/postgresql/oid/money.test.ts
connection-adapters/postgresql/oid/range.test.ts
connection-adapters/postgresql/oid/type-map-initializer.test.ts
connection-adapters/postgresql/oid/uuid.test.ts
connection-adapters/postgresql/oid/vector.test.ts
connection-adapters/postgresql/quoting.test.ts
connection-adapters/postgresql/referential-integrity.test.ts
connection-adapters/postgresql/schema-creation.test.ts
connection-adapters/postgresql/schema-definitions.test.ts
connection-adapters/postgresql/schema-dumper.test.ts
connection-adapters/postgresql/schema-statements-class.test.ts
connection-adapters/postgresql/schema-statements.test.ts
connection-adapters/postgresql/temporal-type-parsers.test.ts
connection-adapters/postgresql/type-map-init.test.ts
connection-adapters/quoting-contract.test.ts
connection-adapters/raw-connection-overload.test.ts
connection-adapters/sqlite3-adapter.hash-constructor.test.ts
connection-adapters/sqlite3-adapter.integer-bind.test.ts
connection-adapters/sqlite3-adapter.query-transformers.test.ts
connection-adapters/sqlite3-adapter.transactions.test.ts
connection-adapters/sqlite3-introspection.test.ts
connection-adapters/sqlite3/column.test.ts
connection-adapters/sqlite3/quoting.test.ts
connection-adapters/sqlite3/schema-creation.test.ts
connection-adapters/sqlite3/schema-definitions.test.ts
connection-adapters/sqlite3/schema-dumper.test.ts
connection-adapters/sqlite3/schema-statements.test.ts
```

The rename itself is not the substance — **classification is**. Each file is
either a trails invention (rename to `*.trails.test.ts`) or an _unported_
Rails file whose plain name is what a future port will match on (leave it, and
note the Rails file it will match). `support/**` was unambiguous because
Rails' `test/support/` holds helper sources and no `*_test.rb` at all; this
directory is not, so check each candidate against
`vendor/rails/activerecord/test/` and `pnpm rails:find` before renaming.

## Acceptance criteria

- [ ] Regenerate the candidate list (it drifts as files are ported).
- [ ] Classify every candidate under `connection-adapters` as trails invention or awaiting a
      Rails port; record the Rails counterpart for the latter.
- [ ] Rename only the trails-invention set. Do NOT touch any test name —
      renames are file-level only.
- [ ] `pnpm parity:test --package activerecord` totals byte-identical before
      and after.
- [ ] Update every hard-coded path that names a renamed file: vitest include
      globs, CI filters, `eslint/*.mjs` allowlists (see
      `eslint/test-infra-scope.mjs`), `scripts/non-transactional-row-writes.json`,
      and source-comment citations.
