---
title: "nested build raises from _assign_attribute, not a bespoke known-key helper"
status: draft
updated: 2026-08-21
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 90
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`assertNestedAttributesAreKnown` in
`packages/activerecord/src/nested-attributes.ts` is a trails-only helper: it
reads `targetModel.attributeTypes()`, exempts the primary key, then falls back
to constructing a probe record and calling `hasAttribute` for alias resolution,
raising `UnknownAttributeError` itself.

Rails has no such helper. The nested build is
`association.build(attributes)` → `assign_attributes`, and the raise comes from
`_assign_attribute` (`activemodel/lib/active_model/attribute_assignment.rb:67-76`):
it `public_send`s the `#{k}=` setter and, on `NoMethodError` with no such
writer, calls `attribute_writer_missing`, whose ActiveRecord override raises
`UnknownAttributeError`. The existing JSDoc says the helper exists because
trails' `Model.new` / build silently drops unknown keys.

## Converged shape

Make the nested build's `assign_attributes` raise the way Rails' does — i.e. fix
the leniency at `_assign_attribute` / `attribute_writer_missing` — and delete
the helper, its probe construction and its PK exemption. The alias resolution
the probe provides is already `_assign_attribute`'s job through the generated
alias writer.

## Acceptance criteria

- [ ] `assertNestedAttributesAreKnown` is deleted; the nested build raises from
      the shared attribute-assignment path.
- [ ] The error class, message and raise site match Rails
      (`attribute_assignment.rb:67-76` + AR's `attribute_writer_missing`).
- [ ] `nested-attributes*` suites pass with no test-name changes.
