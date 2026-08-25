---
title: "Migration's _recording flag is a branch Rails does not have (recorder should BE the connection)"
status: in-progress
updated: 2026-08-25
rfc: "0051-migration-schema-statements-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 400
priority: null
pr: 7038
claim: "2026-08-25T14:18:30Z"
assignee: "migration-recording-flag-should-be-the-connection"
blocked-by: null
closed-reason: null
---

# `Migration`'s `_recording` flag is a branch Rails does not have

## Context

Surfaced converging `Migration#run` onto the Rails signature in PR #7031
(RFC 0051, `migration-run-takes-migration-classes-variadic`).

Rails routes a recorded migration through the **connection**, not through a
flag. `Migration#revert` swaps the recorder in as the connection and swaps it
back (`vendor/rails/activerecord/lib/active_record/migration.rb:857-864`):

```ruby
recorder = command_recorder
@connection = recorder
suppress_messages do
  connection.revert(&block)
end
@connection = recorder.delegate
recorder.replay(self)
```

Because `connection` _is_ the `CommandRecorder` for the duration, every schema
statement in `Migration` is written once — `connection.create_table(...)` — and
recording is invisible to the body. `Migration#run` (migration.rb:937-949)
correspondingly has exactly two arms: the `reverting?` wrap and the per-class
`exec_migration` loop.

trails instead carries a private `_recording` boolean plus a `_recorder` field
and branches on it at the top of roughly **35** methods in
`packages/activerecord/src/migration.ts` (`createTable`, `addColumn`,
`addIndex`, `changeColumn`, … — every `if (this._recording)` site), and `run`
carries a third arm Rails does not have that hand-copies `_recorder`,
`_recording` and `_connectionOverride` onto the sub-migration and restores them
in a `finally`. #7031 moved that arm inside Rails' per-class loop so the
surrounding shape matches, but the arm itself is still trails-only surface.

## Converged shape

- `Migration#revert` assigns the `CommandRecorder` to the connection slot
  (`_connectionOverride`, the existing `@connection` mirror) rather than
  setting a `_recording` flag, and restores `recorder.delegate` afterwards —
  migration.rb:857-864.
- Every `if (this._recording) { this._recorder.foo(...) } else { ... }` pair in
  `migration.ts` collapses to the single `this.connection.foo(...)` call Rails
  writes.
- `Migration#run` loses its third arm entirely: a sub-migration constructed
  inside a recording parent inherits the recorder because it reads the same
  connection, exactly as Rails does.
- `_recording` and `isReverting()`'s dependence on it go away; `reverting?`
  becomes Rails' `connection.respond_to?(:reverting) && connection.reverting`
  (migration.rb:871-873).

## Acceptance criteria

- [ ] No `_recording` field on `Migration`; no `if (this._recording)` branch
      remains in `packages/activerecord/src/migration.ts`.
- [ ] `Migration#run` has exactly Rails' two arms.
- [ ] `isReverting()` reads the connection, per migration.rb:871-873.
- [ ] `invertible-migration.test.ts` and `migration/command-recorder*.test.ts`
      stay green on SQLite, PostgreSQL and MySQL/MariaDB.
