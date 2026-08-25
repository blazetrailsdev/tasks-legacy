---
title: "belongs_to default hoisted ahead of before_validation queue (sync-chain deviation)"
status: ready
updated: 2026-07-27
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activemodel"
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 80
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
---

## Context

PR #4305 (belongs-to-default-with-required-before-validation) made
`belongsTo(name, { default, optional: false })` save cleanly by resolving the
possibly-async default block in an async pre-validation phase
(`Base#_runBelongsToDefaults`, `packages/activerecord/src/base.ts`) invoked from
`save` before `performValidations` (`packages/activerecord/src/persistence.ts`),
plus a synchronous `before_validation` callback registered by
`BelongsTo.addDefaultCallbacks`
(`packages/activerecord/src/associations/builder/belongs-to.ts`).

Deviation (raised 3× in Codex review on #4305): Rails registers the default as a
normal `before_validation` callback
(`activerecord/lib/active_record/associations/builder/belongs_to.rb:103-106` via
`activemodel/lib/active_model/validations/callbacks.rb:55-60`) calling
`BelongsToAssociation#default`
(`activerecord/lib/active_record/associations/belongs_to_association.rb:46-48`)
at its queue position. trails hoists the awaited default ahead of the entire
`before_validation` queue, so a user `before_validation` callback registered
before/after `belongsTo` cannot observe Rails' relative ordering with the
default (it always sees the post-default state on the save path).

Original root cause: the default block may be async (`() => Developer.first()`)
and trails' validation callback chain was strictly synchronous, so an awaited
default could not run at its queue position and was hoisted instead.

**That premise is stale as of RFC 0063 (validations made async).** On
`origin/main` today `isValid()` returns `Promise<boolean>`
(`packages/activemodel/src/validations.ts:123`) and `runValidationsBang` awaits
`this._runValidateCallbacks()` (`validations.ts:306-309`) — the validation
callback chain accepts a Promise, so there is no longer a language-level
obstacle to running the default at its registered `before_validation` position.
The hoist itself is still live (`Base#_runBelongsToDefaults`,
`packages/activerecord/src/base.ts:3848`, invoked from
`packages/activerecord/src/persistence.ts:748-750` with the
`_belongsToDefaultsApplied` sentinel at `:750,:758`), which is why this story
stays open — but it is now ordinary convergence work, not architecture-blocked.

## Acceptance criteria

- The belongs_to `default` block runs at its registered `before_validation`
  queue position relative to user-defined `before_validation` callbacks,
  matching Rails — i.e. a user callback registered before `belongsTo` runs
  before the default; one registered after runs after.
- Remove the hoisted `_runBelongsToDefaults` pre-pass and the
  `_belongsToDefaultsApplied` sentinel once the default runs in-queue.
- Do not ratify the hoist. (The former BLOCKED-ON-ARCHITECTURE caveat — "requires
  the strictly-synchronous validation chain to be revisited" — no longer applies:
  RFC 0063 made the validation chain async. Check the state of
  `async-before-validation-sync-chain` before starting, but this story does not
  depend on it.)
