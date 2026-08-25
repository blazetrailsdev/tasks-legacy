---
title: "Converge new-record scope seeding onto populateWithCurrentScopeAttributes"
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

Rails seeds new records with scope attributes in exactly one place:
`Scoping#populate_with_current_scope_attributes`
(`vendor/rails/activerecord/lib/active_record/scoping.rb:47-51`), called from
`initialize_internals_callback` (`scoping.rb:54-57`).

trails has that method — `populateWithCurrentScopeAttributes`
(`packages/activerecord/src/scoping.ts:117`), mixed onto `Base` at
`base.ts:5009` — but nothing on the construction path calls it. `Base` instead
runs its own `_applyScopeAttributes` (`base.ts:672`, called from `base.ts:3200`
and `base.ts:3283`), which duplicates the logic with an inverted merge: Rails
populates BEFORE `super` so explicit attrs overwrite scope attrs, whereas trails
applies scope attrs after construction and therefore skips keys already present
in the explicit set (`explicitKeys`).

Discovered while porting `scope_attributes?` (#5175): both bodies had to be kept
in step, and only the `Base` copy was on the live path.

## Acceptance criteria

- One body seeds new records: `populateWithCurrentScopeAttributes` is on the
  live construction path, or `_applyScopeAttributes` delegates to it.
- The explicit-attrs-win ordering is preserved (Rails' before-`super` call), with
  the deviation justified at the call site if the inversion must stay.
- Existing scoping/default-scoping/base suites pass unchanged.
