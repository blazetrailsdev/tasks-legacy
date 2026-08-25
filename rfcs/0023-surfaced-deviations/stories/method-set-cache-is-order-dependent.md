---
title: "MethodSet's global cache makes attribute-method generation order-dependent"
status: draft
updated: 2026-08-18
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activesupport"
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

`MethodSet`'s cache is a process-global singleton keyed by namespace —
`METHOD_CACHES = Hash.new { |h, k| h[k] = Module.new }`
(`vendor/rails/activesupport/lib/active_support/code_generator.rb:8`, ported at
`packages/activesupport/src/code-generator.ts:26`). Rails shares it too, but in
Rails a cached entry is a compiled method under a _canonical_ name, so sharing is
safe by construction: `define_cached_method` skips the block when the cache
already defines that name, and `apply` copies the descriptor onto each owner.

trails now generates attribute readers/writers through that cache from the
`define_method_#{proxy_target}` hooks (PR #6717), which means the set of names
in a cache is a function of which model classes a process has defined and in
what order. PR #6717 surfaced this concretely: a composite-PK regression
reproduced only in the full-suite run, never when the two failing files were run
alone or together, because the failing generation depended on class-registration
order across the whole process rather than on anything in those files.

An order-dependent generation path is a latent flake source for every future
attribute-methods change: a test can pass alone and fail in the suite, or the
reverse, with no signal about why.

## Acceptance criteria

- [ ] Determine whether trails' cache keys are canonical in the Rails sense —
      i.e. whether two classes reaching the same `namespace` + canonical name can
      ever want different descriptors (attribute name mangling, `as:` aliases,
      composite-PK `id`).
- [ ] If they can, converge onto Rails' invariant (make the key carry what makes
      the descriptor unique) rather than adding a trails-only cache-busting layer.
- [ ] A test that pins the ordering property directly: define two model classes
      whose attribute names collide in one namespace, in both orders, and assert
      the generated accessors are correct in both.
