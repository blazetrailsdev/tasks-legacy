---
title: "converge-module-ext-generated-method-locations"
status: draft
updated: 2026-08-13
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activesupport"
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced by PR #6456 (RFC 0098 deprecation port). `caller_locations` entered the
call-parity population when `deprecation.ts` gained a member of that name, which
exposed a pre-existing omission in `module-ext.ts`.

Rails threads the declaring source location into every method it generates:

- `activesupport/lib/active_support/core_ext/module/delegation.rb:160-165` —
  `module_eval(method_def, file, line)` built from `caller_locations(1, 1).first`,
  so a `DelegationError` / NameError raised inside the generated method points at
  the `delegate` call, not at delegation.rb.
- `activesupport/lib/active_support/core_ext/module/attribute_accessors.rb:208-211`
  — `mattr_accessor` passes `location: caller_locations(1, 1).first` down to
  `mattr_reader` / `mattr_writer` for the same reason.

trails' `delegate` / `mattr_accessor` generate real JS functions, which carry a
real stack, and there is no `module_eval` file/line to attribute — but the
raised errors still do not name the declaring site the way Rails' do.

Two rows in `scripts/api-compare/call-mismatches-exclude/activesupport/module-ext.json`
(`delegate` → `caller_locations`, `mattr_accessor` → `caller_locations`) carry
this reason and are deleted by this story.

## Acceptance criteria

- Errors raised from a generated `delegate` / `mattr_accessor` member identify
  the declaring call site, by whatever mechanism TS affords (an explicit
  `Error.captureStackTrace` frame or a message suffix — decide against what the
  existing DelegationError tests assert), or the story is blocked with the
  specific reason JS cannot express it.
- Both `caller_locations` rows deleted from the exclude baseline (only-shrink;
  delete by hand, no `--write`).
