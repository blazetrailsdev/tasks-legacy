---
title: "Mirror Rails' ProtectedParams stub shape (private store + delegation)"
status: ready
updated: 2026-07-27
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
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

`ProtectedParams` (packages/activerecord/src/test-helpers/protected-params.ts)
diverges structurally from the Rails stub it mirrors
(`vendor/rails/activerecord/test/support/stubs/strong_parameters.rb`):

- Rails stores parameters in `@parameters` (a `HashWithIndifferentAccess`) and
  `delegate :keys, :key?, :has_key?, :empty?, to: :@parameters`. trails assigns
  them as **own enumerable properties** on the instance, so `Object.keys` /
  `Object.entries` see them directly.
- Rails defines `each_pair`, `to_unsafe_h`, `[]`, and a `dup` that preserves
  `@permitted`. trails has none of these.

Consequence: the stub does not exercise the code paths that exist _because_ real
params wrappers hide their contents. `isMassAssignmentEmpty` has a dedicated
wrapper-`empty` delegation branch (added by
`mass-assignment-empty-bag-delegates-to-wrapper-empty`) that ProtectedParams
never reaches, since its own-props layout makes the plain `Object.keys` fallback
accidentally correct.

This shape difference is why the getter bug fixed in #4972 went undetected: the
AR suite's only params stand-in behaves like a plain hash. (The `permitted?`-as-
method shape is NOT a divergence — Rails' stub defines it as a method too, and
trails correctly mirrors that; the real `ActionController::Parameters` getter is
covered separately in actionpack.)

## Acceptance criteria

- [ ] `ProtectedParams` stores parameters in a private field and exposes
      `keys` / `key?` / `empty?` delegates, matching the Rails stub's shape.
- [ ] Port the missing `each_pair`, `to_unsafe_h`, `[]`, and permitted-preserving
      `dup` members.
- [ ] `forbidden-attributes-protection.test.ts` still passes (16 Rails-mapped
      tests) — adjust the helper, never the Rails-matched test names.
- [ ] The wrapper-`empty` delegation branch in `isMassAssignmentEmpty` is
      genuinely exercised by at least one AR test.
