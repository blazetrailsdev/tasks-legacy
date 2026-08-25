---
title: "CommandRecorder commands carry a callable third element, as in Rails"
status: draft
updated: 2026-07-30
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 140
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced by PR #5635 (`change-table-bulk-paths-use-real-command-recorder`),
which converged both `bulk: true` paths onto the real `CommandRecorder` but
left one deviation deliberately out of scope.

Rails' `CommandRecorder#change_table` stores a **callable** third element in
the command tuple
(`vendor/rails/activerecord/lib/active_record/migration/command_recorder.rb:142`):

```ruby
@commands << [:change_table, [table_name], -> t { bulk_change_table(table_name, commands) }]
```

and `replay` destructures all three, passing the block through
(`command_recorder.rb:147-149`):

```ruby
def replay(migration)
  commands.each do |cmd, args, block|
    migration.send(cmd, *args, &block)
```

Trails' command shape is `{ cmd: string; args: unknown[] }`
(`packages/activerecord/src/migration/command-recorder.ts:15`) with no block
slot. The bulk branch therefore stores the captured sub-commands as the second
arg — `this._commands.push({ cmd: "changeTable", args: [tableName, sub.commands] })`
(`command-recorder.ts:131`) — and `replay()` carries a special case that
expands them one by one instead of forwarding a block
(`command-recorder.ts:143-156`).

The generated `ReversibleAndIrreversibleMethods` bodies also drop Rails'
`&block` (`command_recorder.rb:124-131` records `record(:create_table, args, &block)`),
so `createTable`'s reversal block is threaded through `args` as a trailing
function today (see `invertDropTable`'s "block rides as a trailing function"
handling at `command-recorder.ts:194-199`).

This is a change to the command tuple and to `replay()` for **every** recorded
command, not just `changeTable` — which is why #5635 scoped it out.

## Acceptance criteria

- The recorded command carries a third, optional callable element (e.g.
  `{ cmd, args, block? }`), mirroring Rails' `[cmd, args, block]`.
- `replay()` forwards the block rather than special-casing bulk `changeTable`;
  the `cmd === "changeTable" && Array.isArray(args[1])` branch is deleted.
- `CommandRecorder#changeTable`'s bulk branch stores the parent entry with a
  callable that calls `bulkChangeTable(tableName, commands)`, matching
  `command_recorder.rb:137-143`.
- The generated recordable methods accept and record a trailing block, so
  `invertDropTable`/`invertCreateTable` no longer sniff `args` for a trailing
  function.
- `migration/command-recorder.test.ts`, `migration/change-table.test.ts` and
  `invertible-migration.test.ts` stay green on sqlite3, PostgreSQL and MySQL
  (MySQL is the only adapter where `supportsBulkAlter?` is true).
- `parity:api` / `parity:test` deltas non-negative.
