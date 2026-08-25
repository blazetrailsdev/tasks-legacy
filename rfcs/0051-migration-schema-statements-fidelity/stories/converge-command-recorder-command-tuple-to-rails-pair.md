---
title: "inverseOf and _commands carry Rails' [method, args] pair, not a { cmd, args } object"
status: done
updated: 2026-08-25
rfc: "0051-migration-schema-statements-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 260
priority: null
pr: 7020
claim: "2026-08-25T00:30:08Z"
assignee: "relocate-model-name-to-naming-module"
blocked-by: null
closed-reason: null
---

## Context

Deferred out of `converge-command-recorder-inverse-of-message-and-decomposition`
(PR #7012), which folded `_dispatchInvert` back into `inverseOf` and converged the
`IrreversibleMigration` message. The return shape was the one deviation left
standing, and the PR body says so explicitly.

`vendor/rails/activerecord/lib/active_record/migration/command_recorder.rb:114-123`

```ruby
def inverse_of(command, args, &block)
  method = :"invert_#{command}"
  raise IrreversibleMigration, <<~MSG unless respond_to?(method, true)
    ...
  MSG
  send(method, args, &block)
end
```

Each `invert_*` returns the `[method, args]` pair (e.g.
`command_recorder.rb:200`, `[:drop_table, args]`), and `inverse_of` passes it
straight through. Callers destructure it: `record` pushes it onto `@commands`
(`command_recorder.rb:106-111`), and `replay`/`send_command` splat it back apart.

trails (`packages/activerecord/src/migration/command-recorder.ts`) instead wraps
the pair into an object:

```ts
const [invertedCmd, invertedArgs] = (this[method] as ...).call(this, args);
return { cmd: invertedCmd, args: invertedArgs };
```

That `{ cmd, args }` object is trails' command-tuple representation everywhere —
`_commands`, `record`, `revert`, `inverse`, `replay`, `changeTable`'s bulk path —
so `inverse_of` is only the visible edge of it. The `invert_*` methods themselves
already return the Rails `[string, unknown[]]` pair; it is the tuple that leaves
`inverseOf` and the container it lands in that diverge.

## Converged shape

`inverseOf` returns the `[method, args]` pair Rails returns, and `_commands`
holds pairs rather than `{ cmd, args }` objects, so `record`, `revert`,
`inverse`, `replay` and `changeTable` destructure the same way Rails' callers do.

## Acceptance criteria

- [ ] `inverseOf` returns `[method, args]`, matching command_recorder.rb:122.
- [ ] `_commands` entries are the same pair shape Rails' `@commands` holds
      (command_recorder.rb:106-111).
- [ ] Every internal consumer (`record`, `revert`, `inverse`, `replay`,
      `changeTable` bulk path) reads the pair rather than the object.
- [ ] `command-recorder.test.ts` (~100 `inverseOf` assertions),
      `command-recorder.trails.test.ts` and `invertible-migration.test.ts` green.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.

## Notes

This is a wide but mechanical change — the assertion count in
`command-recorder.test.ts` is what makes it its own story rather than a rider on
the message/decomposition convergence.
