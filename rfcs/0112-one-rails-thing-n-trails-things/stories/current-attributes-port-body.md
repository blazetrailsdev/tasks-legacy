---
title: "Port CurrentAttributes' body to current_attributes.rb (defaults, Delegation.generate, IsolatedExecutionState, reset)"
status: in-progress
updated: 2026-08-25
rfc: "0112-one-rails-thing-n-trails-things"
cluster: duplicate-bodies
packages: []
deps: []
deps-rfc: []
est-loc: 400
pr: 7055
claim: "2026-08-25T17:22:41Z"
assignee: "current-attributes-port-body"
blocked-by: null
closed-reason: null
---

## Context

`packages/activesupport/src/current-attributes.ts` now threads
`ActiveSupport::CodeGenerator` through `attribute`
(story `activesupport-current-attributes-code-generator`, PR #6538), but the
rest of the class is still a trails design rather than a port of
`vendor/rails/activesupport/lib/active_support/current_attributes.rb`.

Standing gaps, with the Ruby each one corresponds to:

- **`attribute`'s signature.** Rails is
  `def attribute(*names, default: NOT_SET)` (`:114`) with a `NOT_SET`
  sentinel and `self.defaults = defaults.merge(names.index_with { default })`
  (`:140`). trails sniffs a trailing options object per call and stores a
  per-name `{ default }` record in a bespoke `_definitions` map, so `default:
nil` and "no default" are indistinguishable and there is no `defaults`
  class attribute.
- **`Delegation.generate`.** Rails generates the singleton readers/writers
  that make `Current.user` work (`:137-138`). trails has
  `_setupProxy()`, an explicit no-op stub, and tells callers to use
  `Current.instance().user` instead.
- **`current_instances` / `current_instances_key`** (`:170-176`) key the
  singleton off `IsolatedExecutionState`, so the instance is per-fiber. trails
  uses a module-level `WeakMap` keyed by the class, so there is no execution
  isolation at all — the docstring at the top of the file describes
  AsyncLocalStorage behaviour the code does not have.
- **`reset` / `resolve_defaults` / `attributes=`** (`:186-206`). Rails'
  `reset` assigns `self.attributes = resolve_defaults`, re-evaluating Proc
  defaults and `dup`-ing value defaults. trails clears `_attributes` and
  re-resolves lazily on read, which differs whenever a default is a mutable
  object.
- **`INVALID_ATTRIBUTE_NAMES`** (`:106`) vs trails' `RESTRICTED_NAMES`, and
  the `ArgumentError` Rails raises vs the plain `Error` trails throws.
- **`_get` / `_set`** have no Rails counterpart; Rails' generated body is
  `attributes[:name]` against the `attributes` accessor.

Surfaced while porting the code-generator half (PR #6538, RFC 0096).

## Converged shape

A method-by-method port of current_attributes.rb: `NOT_SET`, the `defaults`
class attribute, `Delegation.generate` for the singleton surface,
`IsolatedExecutionState`-backed `current_instances`, and
`reset`/`resolve_defaults`/`attributes=` as written. `_definitions`, `_get`,
`_set`, `_instances` and `_setupProxy` all disappear.

Likely more than one PR; split by the bullets above if so.

## Acceptance criteria

- [ ] `attribute` takes `(...names, { default = NOT_SET })` and writes through
      a `defaults` class attribute (current_attributes.rb:114-140).
- [ ] `reset` goes through `resolve_defaults` and `attributes=`
      (`:186-206`), re-evaluating Proc defaults and duplicating value ones.
- [ ] The singleton is `IsolatedExecutionState`-scoped per `:170-176`, and the
      file's header docstring matches what the code does.
- [ ] `_setupProxy` (an explicit no-op stub) is deleted, with the singleton
      accessors generated instead.
- [ ] `pnpm parity:api --package activesupport` improves and
      `pnpm parity:api:extra --package activesupport` loses the
      `_definitions` / `_get` / `_set` / `_setupProxy` novel names.
- [ ] `pnpm vitest run packages/activesupport/src/current-attributes.test.ts`
      passes, with its 2 skipped tests enabled.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
