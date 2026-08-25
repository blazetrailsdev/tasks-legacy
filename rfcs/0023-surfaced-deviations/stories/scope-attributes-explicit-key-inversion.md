---
title: "Audit _applyScopeAttributes' explicit-key inversion against Rails"
status: draft
updated: 2026-08-01
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 150
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`_applyScopeAttributes` (`packages/activerecord/src/base.ts:640`) deliberately
inverts Rails: it writes scope attributes only for keys NOT in the explicit
constructor attrs. The comment above `_shouldApplyScopeAttributes`
(base.ts:563) justifies this with "Rails calls
populate_with_current_scope_attributes BEFORE super (so explicit attrs
overwrite scope attrs). In TS we call it after super, so we invert".

The first clause is misleading about the MRO. `populate_with_current_scope_attributes`
runs inside `Scoping#initialize_internals_callback` (`scoping.rb:53-56`), which
`Core#initialize` calls at `core.rb:475` — before its own `super` at
`core.rb:477`, yes, but the ordering that matters for the STI column is the
unwind of the `initialize_internals_callback` chain itself, which PR #5830
showed the port had backwards (fixed there). The explicit-attrs axis was never
audited against Rails end to end, and the invented skip-explicit-keys filter
has no Rails counterpart: Rails simply `_assign_attributes(scope_attributes)`
and lets ActiveModel's later assignment of the constructor attrs win.

## Acceptance criteria

- Trace, against `core.rb:471-482`, `scoping.rb:44-56` and ActiveModel's
  `initialize`, whether the port's explicit-key filter produces the same
  outcome as Rails for: explicit attr also in scope, scope-only attr,
  multiparameter keys, and store-accessor keys.
- Converge `_applyScopeAttributes` to Rails' ordering where the filter diverges,
  or record at the call site exactly which Rails behaviour the inversion
  reproduces and why the filter is required in the port's call order.
- Correct the base.ts:563 comment's account of Rails' order either way.
- Existing scoping / STI coverage stays green; any TS-only test pinning the
  invented filter is converged, not preserved.
