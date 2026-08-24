---
title: "sweep-trails-only-test-files-remaining"
status: claimed
updated: 2026-08-24
rfc: "0078-sti-schema-reflection-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: null
claim: "2026-08-24T03:03:39Z"
assignee: "sweep-trails-only-test-files-remaining"
blocked-by: null
closed-reason: null
---

## Context

Final slice of RFC 0078 `sweep-trails-only-test-files-onto-trails-name`, whose
first slice (`support/**`, 27 files) landed in trails#6932. The remaining
candidates are covered by `sweep-trails-only-test-files-connection-adapters`
(68), `…-associations` (47) and `…-relation` (15); this story is the **81 that
sit at `packages/activerecord/src/` top level or in a directory with fewer than
ten candidates**. Regenerate with the story's method: extract the TS test
manifest via `pnpm parity:test --package activerecord`, then take every
`packages/activerecord/src/**/*.test.ts` in
`scripts/test-compare/output/ts-tests.json` that never appears as a matched TS
counterpart in the per-file report.

```text
adapters/mysql2/bigint-roundtrip.test.ts
adapters/mysql2/dbconsole.test.ts
adapters/postgresql/adapter.test.ts
adapters/postgresql/dbconsole.test.ts
adapters/sqlite3/bigint-roundtrip.test.ts
adapters/sqlite3/dbconsole.test.ts
adapters/sqlite3/sqlite3-trails-root.test.ts
ar-config.test.ts
association-cache.test.ts
asynchronous-queries.test.ts
attribute-methods/query.test.ts
attribute-methods/time-zone-conversion.test.ts
attribute-methods/write.test.ts
bigint-roundtrip.test.ts
cases/validations-repair-helper.test.ts
coders/column-serializer.test.ts
coders/yaml-column.test.ts
column-names-sync-virtual-exclusion.test.ts
database-configurations/connection-url-resolver.test.ts
delegate.test.ts
delegated-type-scope.test.ts
deprecator.test.ts
destroy-association-async-job.test.ts
encryption-hooks.test.ts
encryption/concurrency.test.ts
encryption/encrypted-attribute-type.test.ts
encryption/extended-deterministic-uniqueness-validator.test.ts
establish-connection.test.ts
fixtures.test.ts
inheritance-namespaced.test.ts
instantiate-schema-types.test.ts
lazy-schema-reflection.test.ts
marshal-serialization.test.ts
message-pack.test.ts
migration/join-table.test.ts
model-codegen.test.ts
model-schema-columnshash-recovery.test.ts
model-schema-load.test.ts
model-schema-sync-load.test.ts
model-schema.test.ts
naked-fixtures.test.ts
ordered-options.test.ts
pooled-test-adapter.test.ts
query-logs-instance.test.ts
query-transformers.test.ts
querying-methods-delegation.test.ts
querying.test.ts
register-model-batch.test.ts
relation-exec-main-query.test.ts
reload-models.test.ts
runtime-registry.test.ts
sanitization-quoter.test.ts
schema-loading.test.ts
scoping/all-queries-option.test.ts
sql-default.test.ts
sqlite-adapter.test.ts
sqlite/better-sqlite3.test.ts
sqlite/expo-sqlite.test.ts
sqlite/libsql.test.ts
sqlite/node-sqlite.test.ts
sqlite/statement-reader.test.ts
strict-loading-sync-reader.test.ts
tasks/sqlite-database-tasks.test.ts
test-fixtures.test.ts
test-fixtures/fixture-connection.test.ts
test-fixtures/use-transactional-tests.test.ts
test-fixtures/with-transactional-fixtures.test.ts
test-helper.test.ts
test-helpers/fixtures/fixtures.test.ts
testing/method-call-assertions.test.ts
testing/sql-capture.test.ts
timestamp-alias-resolution.test.ts
trailtie.test.ts
trailties/controller-runtime.test.ts
trailties/job-runtime.test.ts
transaction.test.ts
type-virtualization/resolve-target.test.ts
type-virtualization/virtualize.test.ts
type/decimal-without-scale.test.ts
type/json.test.ts
yaml-serialization.test.ts
```

The rename itself is not the substance — **classification is**. Each file is
either a trails invention (rename to `*.trails.test.ts`) or an _unported_ Rails
file whose plain name is what a future port will match on (leave it, and note
the Rails file it will match). This slice is the riskiest of the four: the
top-level names are the ones most likely to be awaiting a Rails port, so check
each candidate against `vendor/rails/activerecord/test/` and `pnpm rails:find`
before renaming. Split into more than one PR if the LOC ceiling demands it.

Note `support/load-schema-helper.test.ts` was deliberately left alone in #6932:
a `.trails.` sibling already exists under the same stem, so the two need
reconciling rather than a blind rename.

## Acceptance criteria

- [ ] Regenerate the candidate list (it drifts as files are ported).
- [ ] Classify every remaining candidate as trails invention or awaiting a Rails
      port; record the Rails counterpart for the latter.
- [ ] Rename only the trails-invention set. Do NOT touch any test name.
- [ ] `pnpm parity:test --package activerecord` totals byte-identical before and
      after each rename PR.
- [ ] Update every hard-coded path that names a renamed file: vitest include
      globs, CI filters, `eslint/*.mjs` allowlists, `scripts/*.json` registries,
      and source-comment citations.
- [ ] Resolve `support/load-schema-helper.test.ts` vs its `.trails.` sibling.
