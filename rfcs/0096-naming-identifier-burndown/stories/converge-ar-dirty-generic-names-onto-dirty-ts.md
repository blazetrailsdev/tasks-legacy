---
title: "converge-ar-dirty-generic-names-onto-dirty-ts"
status: done
updated: 2026-08-23
rfc: "0096-naming-identifier-burndown"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: 6893
claim: "2026-08-22T23:01:54Z"
assignee: "converge-ar-dirty-generic-names-onto-dirty-ts"
blocked-by: null
closed-reason: null
---

# Converge the AR dirty predicate names onto `attribute-methods/dirty.ts`

## Context

PR #6821 moved the ActiveRecord-only dirty generics off `ActiveModel::Model`
and collapsed their bodies onto the ports in
`packages/activerecord/src/attribute-methods/dirty.ts`. What is left is the
NAME.

`Base` (`packages/activerecord/src/base.ts`) still carries
`savedChangeToAttribute` and `willSaveChangeToAttribute` as boolean predicates,
while `attribute-methods/dirty.ts` carries the same predicates correctly named
`isSavedChangeToAttribute` / `isWillSaveChangeToAttribute` (Rails
`saved_change_to_attribute?` / `will_save_change_to_attribute?`,
`attribute_methods/dirty.rb:86-88`, `138-140`). After #6821 the `Base` methods
are thin — alias resolution plus the enum `type_cast` Rails does inside
`AttributeMutationTracker#changed?` (`attribute_mutation_tracker.rb:44-48`) —
and delegate to the ports, so this is now purely a naming and call-site job.

The name is currently FORCED by the generated half. A generated
`savedChangeToName` reaches its target through `AttributeMethodPattern`'s
derived `${prefix}Attribute${suffix}` join (`attribute_methods.rb:552`,
`packages/activemodel/src/attribute-methods.ts:101`). Ruby distinguishes the
predicate `saved_change_to_attribute?` from the value reader
`saved_change_to_attribute` by a TRAILING `?`; TypeScript's predicate
convention is a LEADING `is`, which no prefix/suffix join can produce. So
`Base` must expose a boolean under the un-prefixed name for the pattern to
find, and that is the blocker to remove.

This is the same root cause as
[[saved-change-to-attribute-values-generated-half]], which owns the other half
(both `saved_change_to_name` and `saved_change_to_name?` camelize to
`savedChangeToName`, so only the predicate is declared today —
`base.ts`'s `attributeMethodPatterns` static block).

## Acceptance criteria

- [ ] `AttributeMethodPattern` can name a proxy target the prefix/suffix join
      does not derive (or the equivalent), so the pattern can dispatch to
      `isSavedChangeToAttribute`.
- [ ] `Base` carries no `savedChangeToAttribute` / `willSaveChangeToAttribute`
      predicate; the alias + enum `type_cast` step lands wherever the ports can
      reach the class.
- [ ] Production call sites move to the `is…` spellings —
      `autosave-association.ts`, `store.ts`, `timestamp.ts`, `persistence.ts`,
      `belongs-to-association.ts`, `belongs-to-polymorphic-association.ts`,
      `base.ts` (optimistic lock), `test-helpers/models/topic.ts`.
- [ ] The unwired `savedChangeToAttribute` value-reader port
      (`attribute-methods/dirty.ts:34`, wrapped at `attribute-methods.ts:968`)
      is either mixed onto `Base.prototype` or deleted — today it is reachable
      from neither, on `main` or on #6821.
- [ ] The shared `changed()` helper in `attribute-methods/dirty.ts` moves onto
      the mutation tracker (`packages/activemodel/src/dirty.ts`), where Rails
      puts `changed?`, so `Model.attributeChanged` /
      `Model.attributePreviouslyChanged` stop repeating the from/to compare.
- [ ] `parity:api` / `parity:test` deltas non-negative; `parity:api:calls` /
      `:args` add zero rows.
