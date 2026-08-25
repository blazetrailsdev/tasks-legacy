---
title: "Verify trails' save/destroy carry Transactions#with_transaction_returning_status semantics"
status: draft
updated: 2026-07-31
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 120
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced by the `DELEGATION_MAX_CALLS` sweep in PR #5738: raising the cap
silences the wide-gate row `base.ts#destroy` missing
`with_transaction_returning_status`, which prompted checking whether the call is
actually made.

Rails wraps both methods in `ActiveRecord::Transactions`
(`vendor/rails/activerecord/lib/active_record/transactions.rb:356-362`):

```ruby
def destroy # :nodoc:
  with_transaction_returning_status { super }
end

def save(**) # :nodoc:
  with_transaction_returning_status { super }
end
```

trails has `withTransactionReturningStatus`
(`packages/activerecord/src/transactions.ts:559`) and wires
`withTransactionReturningStatus` onto the model (`base.ts:5138`), but
`transactions.ts` defines no `destroy` / `save` override at all — `base.ts:4915`
binds `destroy` straight to `_Persistence.destroy`. So the Rails layering where
Transactions decorates Persistence appears absent, and whatever transaction
wrapping trails does happens somewhere else.

Not yet confirmed as a behavioral gap: the wrapping may be equivalent through
another path, in which case this is a layout deviation to justify at the call
site rather than a bug. Verify before porting.

## Acceptance criteria

- Determine whether trails' `save` / `destroy` obtain the
  `with_transaction_returning_status` semantics (status-returning rollback on a
  false return) by some other route, with the trails `file:line` that does it.
- If the semantics are missing, port the `Transactions#destroy` / `#save`
  overrides following the mixin convention, with tests named verbatim after the
  Rails cases in
  `vendor/rails/activerecord/test/cases/transactions_test.rb`.
- If the semantics are present but the layering differs, record the deviation at
  the call site and drop the corresponding wide-gate baseline row.
