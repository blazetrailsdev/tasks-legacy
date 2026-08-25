---
title: "checkValidity should be checkValidityBang (Rails check_validity!)"
status: draft
updated: 2026-08-17
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activemodel"
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

Surfaced by PR #6626 while running `pnpm parity:api:extra --package activemodel`:
`checkValidity` is reported as novel surface in every validator that defines it.

Rails' method is `check_validity!` — a bang method
(`vendor/rails/activemodel/lib/active_model/validations/comparison.rb:11`,
`clusivity.rb:12`, `format.rb:14`, `length.rb:20`, `numericality.rb:23`,
`presence.rb` and `validator.rb`'s `EachValidator#initialize` call site). Per
docs/ruby-ts-conventions.md a Ruby bang method is spelled `...Bang`, so the
converged name is `checkValidityBang`, matching `clearValidatorsBang` and
`mergeBang` elsewhere.

Novel rows measured on the merge commit (one per file):
`validations/comparison.ts`, `validations/exclusion.ts`,
`validations/format.ts`, `validations/inclusion.ts`,
`validations/numericality.ts`, plus `validator.ts`'s declaration and the
`EachValidator` call site. ActiveRecord subclasses that override it
(`validations/uniqueness.ts`, the association `check_validity!` family) need
the same rename in the same pass or they stop overriding.

## Converged shape

Rename `checkValidity` → `checkValidityBang` at every definition, override and
call site; the novel `parity:api:extra` rows disappear because the Ruby name
then resolves. No behavioral change.

## Acceptance criteria

- [ ] `pnpm parity:api:extra --package activemodel` no longer reports
      `checkValidity` as novel in any validator file.
- [ ] `pnpm parity:api` for activemodel does not drop; ActiveRecord's
      overrides renamed in the same pass.
