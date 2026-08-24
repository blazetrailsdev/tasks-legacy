---
title: "inverseOf folds _dispatchInvert back in and raises Rails' IrreversibleMigration message"
status: claimed
updated: 2026-08-24
rfc: "0051-migration-schema-statements-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 90
priority: 11
pr: null
claim: "2026-08-24T23:18:09Z"
assignee: "move-ts-only-extras-out-of-mirrored-activemodel-serialization-test-file"
blocked-by: null
closed-reason: null
---

## Context

Surfaced by PR #6252 while adding the `respond_to?(method, true)` guard.

Rails has no `_dispatch_invert`. `inverse_of` itself does the whole job
(`vendor/rails/activerecord/lib/active_record/migration/command_recorder.rb:114-123`):

```ruby
def inverse_of(command, args, &block)
  method = :"invert_#{command}"
  raise IrreversibleMigration, <<~MSG unless respond_to?(method, true)
    This migration uses #{command}, which is not automatically reversible.
    To make the migration reversible you can either:
    1. Define #up and #down methods in place of the #change method.
    2. Use the #reversible method to define reversible behavior.
  MSG
  send(method, args, &block)
end
```

trails has two deviations here
(`packages/activerecord/src/migration/command-recorder.ts:102-105,666-678`):

1. **Extra decomposition.** A private `_dispatchInvert(cmd, args)` holds the body,
   and both `record` and `inverseOf` call it. Rails' `record` calls `inverse_of`
   directly (`command_recorder.rb:106-111`), so there is one method, not two.
2. **Wrong error message.** The raise says `` `${cmd} is not reversible` ``; Rails
   raises the four-line message above, which names the command _and_ tells the
   author the two ways to make the migration reversible. Anyone reading a real
   migration failure gets strictly less than Rails gives them.

`inverseOf` also returns `{ cmd, args }` where Rails returns the `[method, args]`
pair its callers destructure — worth checking against call sites while in here.

## Converged shape

Fold `_dispatchInvert` back into `inverseOf`, keeping the
`respond_to?(method, true)` membership guard PR #6252 added (a name the recorder
does not answer now reads back as the proxy's NoMethodError-raising function, so
the membership test must precede the read), and raise Rails' message verbatim.

## Acceptance criteria

- [ ] `_dispatchInvert` is gone; `record` and `inverseOf` match
      `command_recorder.rb:106-123` method-for-method.
- [ ] The `IrreversibleMigration` message matches `command_recorder.rb:117-120`
      verbatim, including the two numbered lines.
- [ ] `command-recorder` ported + trails suites and `invertible-migration.test.ts`
      stay green.
