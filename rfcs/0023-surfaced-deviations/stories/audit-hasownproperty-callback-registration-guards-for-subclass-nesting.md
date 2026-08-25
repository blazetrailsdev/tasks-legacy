---
title: "Audit hasOwnProperty-keyed callback registration guards for subclass nesting"
status: ready
updated: 2026-07-27
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

PR 5278 fixed one instance of a class of bug: a callback-registration marker
stored per class with `Object.prototype.hasOwnProperty.call(model, KEY)` does
NOT dedup across the inheritance chain, so a subclass declaring the same kind
of association re-registers the callback and it runs twice on one save.

The instance that was fixed: `_AUTOSAVE_AROUND_SAVE_KEY` in
`packages/activerecord/src/autosave-association.ts`
(`addAutosaveAssociationCallbacks`, the `isCollection` arm). `Firm` declaring
its own collections over `Company`'s registered a second
`aroundSaveCollectionAssociation`, and the nested invocation recomputes
`_newRecordBeforeSave` as `!prev && new_record?` — Rails' own re-entrancy rule
(`autosave_association.rb`, `around_save_collection_association`) — which
clobbered the outer `true` and made `save_collection_association` skip every
already-persisted child of a just-inserted owner. Rails never nests here
because `around_save :around_save_collection_association` is a symbol and
`CallbackChain#append_one` calls `remove_duplicates` before inserting
(`activesupport/lib/active_support/callbacks.rb:644-661`), which dedups across
the chain. The fix was to look the marker up through the static prototype
chain (`KEY in model`).

The same pattern may exist at other registration sites. `defineNonCyclicMethod`
in the same file uses `hasOwnProperty` on `model.prototype`, which is correct
for its purpose (Rails' `define_non_cyclic_method` is also per-class), but
other `hasOwnProperty`-keyed one-time-registration guards have not been audited.

## Acceptance criteria

- Grep the activerecord source for one-time callback/hook registration guards
  keyed with `hasOwnProperty` on a class object or its prototype
  (`autosave-association.ts`, `counter-cache.ts`, `touch-later.ts`,
  `timestamp.ts`, association builders, `nested-attributes.ts`).
- For each, decide from the Rails counterpart whether the dedup is per-class
  (symbol defined per class, e.g. `define_non_cyclic_method`) or chain-wide
  (a plain `around_save :symbol` / `after_save :symbol` that
  `remove_duplicates` collapses) — read the Rails registration site, do not
  infer from the trails code.
- Convert the chain-wide ones to a prototype-chain lookup; leave the
  per-class ones alone with a one-line comment saying which they are.
- Each conversion needs a regression test that fails on baseline: a subclass
  declaring the same association kind as its parent, asserting the callback
  runs once.
