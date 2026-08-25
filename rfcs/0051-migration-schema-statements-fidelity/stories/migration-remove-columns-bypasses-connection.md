---
title: "Migration#removeColumns loops instead of forwarding to the connection"
status: in-progress
updated: 2026-08-25
rfc: "0051-migration-schema-statements-fidelity"
cluster: null
deps: []
deps-rfc: []
est-loc: 50
priority: 18
pr: 7018
claim: "2026-08-25T00:06:07Z"
assignee: "move-ts-only-extras-out-of-mirrored-activemodel-naming-test-file"
blocked-by: null
closed-reason: null
---

## Context

`Migration#removeColumns` (`packages/activerecord/src/migration.ts:1032-1050`)
re-implements column removal as a per-column loop:

```ts
for (const col of columns) {
  await this.removeColumn(tableName, col, opts.type, { ifExists: opts.ifExists });
}
```

Rails' `Migration` has no `remove_columns` of its own — it forwards to the
connection through `method_missing`
(`vendor/rails/activerecord/lib/active_record/migration.rb`, the
`connection.send(method, *arguments, &block)` path), so a migration's
`remove_columns` gets whatever the adapter does.

Since PR #5571, the connection's `remove_columns` issues Rails' single combined
`ALTER TABLE` (`abstract/schema_statements.rb:675-682`) and delegates to
SQLite's single `alter_table` rebuild. The Migration wrapper bypasses both, so
the same call emits N statements from a migration and 1 from the connection.

`test_remove_columns_single_statement` only exercises the connection path
(`connection.remove_columns`), which is why this was not caught. The Migration
path is exercised by `migration.test.ts:1717`.

The recorder branch above it (`this._recording`) is legitimate and must stay —
`CommandRecorder#invert_remove_columns` needs the call recorded as one
`removeColumns` op. Only the non-recording branch should forward.

## Acceptance criteria

- [ ] `Migration#removeColumns` forwards to the connection rather than looping,
      preserving the `_recording` branch unchanged.
- [ ] A test asserting the migration path emits the same single statement the
      connection path does (mirror `test_remove_columns_single_statement`'s
      `assertQueriesCount` shape: 14 on SQLite, 1 elsewhere). Verify it fails on
      baseline.
- [ ] `migration.test.ts` and `migration/command-recorder.test.ts` stay green.
- [ ] Green on all three lanes.
