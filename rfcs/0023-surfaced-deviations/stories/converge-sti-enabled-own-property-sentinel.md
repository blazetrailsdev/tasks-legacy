---
title: "Converge stiEnabled's own-property sentinel onto Rails' column-presence STI gate"
status: ready
updated: 2026-07-27
rfc: "0023-surfaced-deviations"
cluster: null
packages:
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

Surfaced while converging `enableSti` onto the `inheritanceColumn` setter
(#5376). Deleting `enableSti` removed the _spelling_ of the deviation but not
the deviation itself: trails still gates registry-resolved STI dispatch on an
**own-property sentinel**, `stiEnabled(modelClass)` — true only when
`_inheritanceColumn` is an own property of the class, i.e. only when something
explicitly assigned `X.inheritanceColumn = ...`.

Rails has no such gate. `class_attribute :inheritance_column, default: "type"`
(model_schema.rb:172) means every model already answers `"type"`, and STI
participation is decided purely by column presence
(`finder_needs_type_condition?`, inheritance.rb:322) with the autoloader
resolving a row's type name on demand (`find_sti_class`, inheritance.rb:245).

The visible cost is in the test helpers: `test-helpers/models/company.ts:668`
carries `Company.inheritanceColumn = "type"`, a line that does **not** exist in
`vendor/rails/activerecord/test/models/company.rb` (verified — no
`inheritance_column` assignment anywhere in that file). It is there only to trip
the sentinel. `test-helpers/models/post.ts:1007-1010` documents the other side:
Post gets STI via column-presence detection with no assignment at all, so the
two canonical STI hierarchies are opted in by two different mechanisms.

The sentinel is not gratuitous — it picks which of two no-match behaviours
applies (`findStiClassForRow` / `subclassFromAttributesForNew` in
inheritance.ts:1180-1270): a sentinel hierarchy raises `SubclassNotFound` via
the global `findStiClass`, while a merely-column-reflecting base DEGRADES to the
base class, because trails has no autoloader and cannot distinguish an
unloaded-but-valid subclass from a genuinely bad type. Converging it therefore
means deciding how trails substitutes for Ruby's autoloader, not just deleting a
branch.

## Acceptance criteria

- Decide and document the autoloader substitute: either a registry-completeness
  signal that lets every column-reflecting base raise `SubclassNotFound`
  faithfully, or an explicit justification for keeping two arms (recorded at the
  call site, per CLAUDE.md).
- If convergence is chosen: drop `stiEnabled`'s own-property check so STI
  participation is decided by column presence alone, matching
  `finder_needs_type_condition?`.
- Delete `Company.inheritanceColumn = "type"` from
  `test-helpers/models/company.ts` so the helper matches `company.rb` verbatim.
- STI suites pass unchanged: `inheritance.test.ts`,
  `inheritance-namespaced.test.ts`, `sti-attribute-routing.test.ts`, plus
  `associations/eager.test.ts` and `associations/has-many-associations.test.ts`.
- No test renamed; parity:api and parity:test deltas non-negative.
