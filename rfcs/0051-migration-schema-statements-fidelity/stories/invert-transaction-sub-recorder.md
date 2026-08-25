---
title: "invert_transaction throws where Rails reverts the block through a sub-recorder"
status: draft
updated: 2026-08-25
rfc: "0051-migration-schema-statements-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 180
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`invert_transaction` (`vendor/rails/activerecord/lib/active_record/migration/command_recorder.rb:186-194`)
is reversible when it is given a block — it reverts the block through a
sub-recorder and hands back a proc that replays the inversions:

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

`packages/activerecord/src/migration/command-recorder.ts:505-510` instead takes
no `block` and unconditionally throws `IrreversibleMigration`, so
`revert { transaction { ... } }` can never invert in trails where it does in
Rails.

The block seat this needs now exists — PR #7025 (RFC 0051,
`command-recorder-tuple-has-no-block-seat`) converged `_commands` to
`[cmd, args, block]`, and `record` / `inverseOf` thread the block through. That
PR deliberately stopped short of this method because the port is not reachable
without a second half, below.

## The second half: `Migration#transaction`

Rails' `InvertibleTransactionMigration`
(`vendor/rails/activerecord/test/cases/invertible_migration_test.rb:36-42`)
writes a bare `transaction do super end`, which in recording mode reaches
`CommandRecorder#transaction` (it is in `ReversibleAndIrreversibleMethods`) and
is recorded. trails has no `Migration#transaction`: `migration.ts:2767` only
uses `this.connection.transaction(fn)` inside `ddlTransaction`, and the port of
that test class (`invertible-migration.test.ts:54-60`) calls
`this.connection.transaction(...)`, which bypasses the recorder entirely. So
today nothing but a direct `recorder.transaction(...)` in a test can reach
`invertTransaction` at all, and converging it alone would land dead code.

## Acceptance criteria

- `invertTransaction` mirrors `command_recorder.rb:186-194`: takes the block,
  reverts it through a sub-recorder built on `delegate`, and returns
  `["transaction", args, invertionsProc]` where the proc replays the sub-recorder
  into the recorder. trails' `revert` is async, so the eager revert's promise has
  to be awaited inside the proc — a language-forced deviation, justified at the
  call site.
- `Migration#transaction` exists and records through the recorder in recording
  mode, so `transaction { ... }` inside a `change` is reversible.
- `invertible-migration.test.ts`'s `InvertibleTransactionMigration` mirrors the
  Ruby (`this.transaction(...)`, not `this.connection.transaction(...)`), and
  `migrate revert transaction` still passes.
- `command-recorder.test.ts`'s `invert transaction with irreversible inside is
irreversible` still raises — after this change via Rails' path (the
  sub-recorder failing to invert `execute`) rather than the unconditional throw.
- SQLite, PostgreSQL and MySQL/MariaDB lanes green.

## Status after PR #7038

PR #7038 (`migration-recording-flag-should-be-the-connection`) landed most of
this while making the recorder the connection, so the remaining scope is
narrower than the body above:

- DONE — `invertTransaction` no longer throws. It builds
  `["transaction", args, invertionsProc]` over a sub-recorder on `delegate`,
  mirroring `command_recorder.rb:186-190`.
- DONE — `transaction` is recorded: `CommandRecorder.prototype.transaction`
  (the generated forwarder, `command_recorder.rb:125-132`) now exists, and
  `invertible-migration.test.ts` > `migrate revert transaction` passes through
  it, because `Migration`'s connection IS the recorder while reverting.
- DONE — `command-recorder.test.ts` > `invert transaction with irreversible
inside is irreversible` now raises through Rails' path (the block's `execute`
  failing to invert), not an unconditional throw.
- REMAINING — the block is run by the `transaction` forwarder, not inside
  `invertTransaction`, because `inverse_of` is sync in both languages while a
  TS block returns a promise and the run must COMPLETE before the `transaction`
  tuple is appended (Ruby's ordering, `command_recorder.rb:187`). Converging
  this means an await point inside `record`/`inverseOf` — i.e. making that
  chain async — which also reds every synchronous
  `expect(recorder.inverseOf(...)).toEqual(...)` in `command-recorder.test.ts`.
  Weigh that against leaving the split, which is documented at both call sites.
- REMAINING — `Migration#transaction` still does not exist, so
  `InvertibleTransactionMigration` mirrors the Ruby's behaviour through
  `this.connection.transaction(...)` rather than its spelling. Note Rails'
  `Migration` has no `#transaction` either — it reaches the connection through
  `method_missing` (`migration.rb:1045-1057`) — so check whether this criterion
  is asking for a method Rails does not have before adding one.
