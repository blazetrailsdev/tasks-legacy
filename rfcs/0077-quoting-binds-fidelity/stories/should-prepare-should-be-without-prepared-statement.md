---
title: "_shouldPrepare should be Rails' without_prepared_statement? on the abstract adapter"
status: ready
updated: 2026-08-25
rfc: "0077-quoting-binds-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 60
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Rails has no `should_prepare?`. The predicate at this position is
`without_prepared_statement?`, the **inverse**:

```ruby
def without_prepared_statement?(binds)
  !prepared_statements || binds.empty?
end
```

(`vendor/rails/activerecord/lib/active_record/connection_adapters/abstract_adapter.rb:1177`.)

trails spells it as a positive `_shouldPrepare` on both adapters —
`mysql2-adapter.ts` (`private _shouldPrepare(binds)`) and
`postgresql-adapter.ts` — with a leading underscore and an inverted sense, so
neither the name nor the polarity matches the Ruby a Rails dev would look for.
The call sites read `const prepare = ... this._shouldPrepare(binds)` where
Rails reads `if !prepare || without_prepared_statement?(binds)`.

Surfaced by `should-prepare-statement-limit-gate-is-invented` (PR #6293),
which converged the _body_ of both gates to Rails' condition but deliberately
left the name and polarity alone to keep that PR scoped.

## Converged shape

`withoutPreparedStatement(binds)` on `AbstractAdapter` — one definition, at
Rails' position, returning `!this.preparedStatements || binds.length === 0` —
with the per-adapter `_shouldPrepare` copies deleted and the call sites
inverted to match `abstract_adapter.rb:1177`.

Note it is defined **once on the abstract adapter** in Rails, not per adapter;
converging the name should also collapse the two duplicate definitions.

## Acceptance criteria

- [ ] `_shouldPrepare` is gone from both adapters; `withoutPreparedStatement`
      exists once on `AbstractAdapter`.
- [ ] Call sites read the inverted form, matching `abstract_adapter.rb:1177`.
- [ ] `pnpm parity:api:extra --package activerecord` loses the two novel rows.
- [ ] parity:api / parity:test delta non-negative; all three lanes green.
