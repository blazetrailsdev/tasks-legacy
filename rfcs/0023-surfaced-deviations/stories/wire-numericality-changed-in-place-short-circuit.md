---
title: "Wire numericality record_attribute_changed_in_place? short-circuit"
status: ready
updated: 2026-07-05
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activemodel"
deps: []
deps-rfc: []
est-loc: 40
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Rails `NumericalityValidator#prepare_value_for_validation`
(`vendor/rails/activemodel/lib/active_model/validations/numericality.rb:121`)
has an early `return value if record_attribute_changed_in_place?(record, attr_name)`
short-circuit: when an attribute was mutated in place the cast value IS the
user's change, so the stale `before_type_cast` raw must not be preferred.

trails omits this gate — see
`packages/activemodel/src/validations/numericality.ts` `prepareValueForValidation`
(the comment there) and the exported-but-unused
`isRecordAttributeChangedInPlace` (numericality.ts ~582).

Historically it was skipped because `Model.attributeChangedInPlace` returned
true for ANY change. PR #4621 changed `attributeChangedInPlace`
(`packages/activemodel/src/model.ts` ~2447) to infer genuine in-place mutation
(live value diverged from the dirty tracker's recorded change), so wiring the
short-circuit is now plausible.

## Acceptance criteria

- [ ] `prepareValueForValidation` returns the cast `value` (not the raw
      before-type-cast) when `isRecordAttributeChangedInPlace(record, attrName)`
      is true, mirroring numericality.rb:121.
- [ ] Add coverage for the in-place-mutation path (mirror the relevant Rails
      numericality test) confirming a mutated numeric value validates against the
      cast value, and that ordinary `10 -> "abc"` assignments still fail.
- [ ] No regressions in numericality-validation.test.ts (activemodel + activerecord).
