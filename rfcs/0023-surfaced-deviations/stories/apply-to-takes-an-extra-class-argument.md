---
title: "PendingModification#applyTo takes a class argument Rails' apply_to does not"
status: draft
updated: 2026-08-15
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activemodel"
deps: []
deps-rfc: []
est-loc: 80
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced by PR #6558 (RFC 0096 wave-4 naming burndown for activemodel). A
`class: "naming"` call-argument row on
`packages/activemodel/src/attribute-registration.ts` survives after the local
was renamed to the Rails identifier, because trails' `applyTo` takes an argument
Rails' `apply_to` does not.

Rails, `activemodel/lib/active_model/attribute_registration.rb:86-88`:

    pending_attribute_modifications.each do |modification|
      modification.apply_to(attribute_set)
    end

and every `apply_to` implementation takes exactly the one argument — see
`PendingType#apply_to(attribute_set)` and `PendingDecorator#apply_to(attribute_set)`
in the same file, which resolve names off the `attribute_set` they are handed.

trails, `attribute-registration.ts:281-283`:

    for (const modification of collectPendingModifications(cls)) {
      modification.applyTo(attributeSet, cls);
    }

The extra `cls` is threaded so `PendingDecorator` / `PendingType` can resolve
attribute names against the class rather than off the `AttributeSet`. That is
the divergence: Rails' `apply_to` has everything it needs in `attribute_set`.

## Converged shape

`applyTo(attributeSet)` — one argument, on the `PendingModification` interface
and both implementations (`PendingDecorator`, and whatever plays `PendingType`)
— with name resolution taken off the `AttributeSet` as Rails does. The
`class: "naming"` row for `attribute-registration.ts` / `apply_to` then clears
in `pnpm parity:api:calls:args:report` with no new `shape` row.

## Acceptance criteria

- [ ] `applyTo` takes only `attributeSet`, matching attribute_registration.rb:86-88.
- [ ] The `apply_to` naming row clears; no new `shape` row; no baseline row added
      or widened.
- [ ] activemodel + activerecord attribute/type suites green on all three lanes.
