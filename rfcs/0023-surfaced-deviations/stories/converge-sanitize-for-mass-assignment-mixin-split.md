---
title: "Converge the sanitizeForMassAssignment / sanitizeForbiddenAttributes mixin split"
status: draft
updated: 2026-07-28
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activemodel"
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

Rails' `ActiveModel::ForbiddenAttributesProtection` defines
`sanitize_for_mass_assignment` and relies on `include` to override the
`AttributeAssignment` method of the same name
(`vendor/rails/activemodel/lib/active_model/forbidden_attributes_protection.rb:19`).
Callers — including the tests — only ever say `sanitize_for_mass_assignment`
(`vendor/rails/activemodel/test/cases/forbidden_attributes_protection_test.rb:31,37,42`).

TS mixins cannot override that way, so trails split the two into distinct
methods:

- `packages/activemodel/src/attribute-assignment.ts:134` —
  `sanitizeForMassAssignment`
- `packages/activemodel/src/forbidden-attributes-protection.ts:30` —
  `sanitizeForbiddenAttributes`, which delegates to the above

Both are mounted on `Model` (`packages/activemodel/src/model.ts:2457,2467`), and
call sites must pick the right one by hand. `forbidden-attributes-protection.test.ts`
calls `sanitizeForbiddenAttributes` where Rails calls
`sanitize_for_mass_assignment`. Surfaced while reviewing PR #5532.

Separately, the file-local `ProtectedParams` stub in
`packages/activemodel/src/forbidden-attributes-protection.test.ts` omits Rails'
`delegate :keys, :key?, :has_key?, :empty?, to: :@parameters`. Deliberately left
out in #5532 (no test in that file exercises them), but it is a stub-shape
divergence from Rails and from the AR stub
(`packages/activerecord/src/support/stubs/strong-parameters.ts`), which does
carry all four.

## Acceptance criteria

- There is one `sanitizeForMassAssignment` entry point on `Model` whose behavior
  matches Rails' post-`include` composition, or the two-method split is
  documented as a justified, permanent deviation at the definition site.
- If converged: every call site (`model.ts`, `activerecord/src/base.ts`,
  `activerecord/src/persistence.ts`, and the AM/AR tests) goes through the
  Rails-named method, and `forbidden-attributes-protection.test.ts` calls
  `sanitizeForMassAssignment` like Rails does.
- `parity:api` and `parity:test` deltas are non-negative.
