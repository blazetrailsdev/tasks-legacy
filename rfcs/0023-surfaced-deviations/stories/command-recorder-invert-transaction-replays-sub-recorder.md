---
title: "invert_transaction replays a sub-recorder instead of raising unconditionally"
status: draft
updated: 2026-08-07
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 90
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`CommandRecorder#invertTransaction`
(`packages/activerecord/src/migration/command-recorder.ts:471-475`)
unconditionally throws:

```ts
invertTransaction(args: unknown[]): [string, unknown[]] {
  throw new IrreversibleMigration(
    "This migration uses transaction, which is not automatically reversible.",
  );
}
```

Rails does not. `activerecord/lib/active_record/migration/command_recorder.rb:186-195`:

```ruby
def invert_transaction(args, &block)
  sub_recorder = CommandRecorder.new(delegate)
  sub_recorder.revert(&block)

  invertions_proc = proc {
    sub_recorder.replay(self)
  }

  [:transaction, args, invertions_proc]
end
```

A reverted `transaction` block is REVERSIBLE in Rails whenever its contents
are: the block is replayed into a sub-recorder in reverting mode, and the
inverse is a `:transaction` command carrying a proc that replays the
sub-recorder's inverted commands. `IrreversibleMigration` surfaces only from
the _inner_ command that has no inverse, raised by the sub-recorder's own
`record`/`inverse_of`.

Found while porting `test_invert_transaction_with_irreversible_inside_is_irreversible`
(`activerecord/test/cases/migration/command_recorder_test.rb:474-482`) in
PR #6181. That test passes today — but for the wrong reason: it asserts the
raise comes from the irreversible `execute` _inside_ the transaction, and the
trails body raises before ever looking at the block. So the test is currently a
tautology, and the reversible-transaction path has no cover at all.

Both pieces this needs already exist on the class: `revert()` and `replay()`
(`command-recorder.ts:92-105`, `:159-174`).

## Converged shape

- `invertTransaction(args)` builds a sub-recorder over the same `delegate`,
  reverts the block into it, and returns `["transaction", [...args, proc]]`
  where the proc replays the sub-recorder — the trails `{ cmd, args }` analogue
  of Rails' `[:transaction, args, invertions_proc]` triple, with the callable
  riding in `args` as it does elsewhere in this file.
- The unconditional `throw` is deleted. `IrreversibleMigration` then arrives
  from the inner command, which is what the ported test names.
- `test_invert_transaction_with_irreversible_inside_is_irreversible` must still
  pass, and must fail if the sub-recorder is _not_ consulted (i.e. it stops
  being a tautology).

## Acceptance criteria

- `invertTransaction` mirrors `command_recorder.rb:186-195`; no unconditional raise.
- A reverted `transaction` whose body is reversible produces a `transaction`
  command that replays the inverted inner commands.
- `test_invert_transaction_with_irreversible_inside_is_irreversible` stays green
  and is no longer vacuous.
- `parity:api` / `parity:api:calls` non-negative.
