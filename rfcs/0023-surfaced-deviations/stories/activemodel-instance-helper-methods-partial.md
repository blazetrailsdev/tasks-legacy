---
title: "Only 2 of 12 HelperMethods exist as instance methods (Rails includes as well as extends)"
status: draft
updated: 2026-08-17
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activemodel"
deps: []
deps-rfc: []
est-loc: 200
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while porting `test_validate` from `validations_test.rb`
(`vendor/rails/activemodel/test/cases/validations_test.rb:336-347`) in PR #6647.

Rails both `extend`s and `include`s `HelperMethods`
(`vendor/rails/activemodel/lib/active_model/validations.rb:45-46`):

```ruby
extend  HelperMethods
include HelperMethods
```

so every `validates_*_of` helper exists as a **class** method (registering a
validator) _and_ an **instance** method (running that validator on the spot
against `self`). The instance form is what a `validate do ... end` block calls:

```ruby
Topic.validate do
  validates_presence_of :title, :author_name
  validates_length_of :content, minimum: 10
end
```

PR #6647 added the two instance helpers that test needs —
`Model#validatesPresenceOf` and `Model#validatesLengthOf`
(`packages/activemodel/src/model.ts`, next to the instance
`Model#validatesWith`) — and stopped there, since the remaining ten are not
exercised by `validations_test.rb`.

Still missing on the instance side, each defined in `HelperMethods` inside its
own validator file:

- `validates_absence_of` — `validations/absence.rb`
- `validates_numericality_of` — `validations/numericality.rb`
- `validates_inclusion_of` — `validations/inclusion.rb`
- `validates_exclusion_of` — `validations/exclusion.rb`
- `validates_format_of` — `validations/format.rb`
- `validates_acceptance_of` — `validations/acceptance.rb`
- `validates_confirmation_of` — `validations/confirmation.rb`
- `validates_comparison_of` — `validations/comparison.rb`
- `validates_length_of`'s alias `validates_size_of` — `validations/length.rb:127`
- `validates_each` — `validations.rb:161`

Each is a one-liner of the same shape as the two that landed:
`validates_with <X>Validator, _merge_attributes(attr_names)`, where the instance
`validates_with` (`validations/with.rb:143-151`, ported at
`packages/activemodel/src/model.ts` as the async instance `validatesWith`)
constructs and runs the validator immediately rather than registering it.

## Converged shape

For each name above, an instance method beside the existing two, e.g.

```ts
/** Mirrors: HelperMethods#validates_absence_of (validations/absence.rb:31-33). */
async validatesAbsenceOf(...attrNames: unknown[]): Promise<void> {
  await this.validatesWith(
    AbsenceValidator,
    (this.constructor as typeof Model)._mergeAttributes([...attrNames]),
  );
}
```

`async` because trails' instance `validatesWith` is async (RFC 0063 made the
validation chain async); Rails' is synchronous.

## Acceptance criteria

- All ten remaining `HelperMethods` members exist as instance methods on `Model`,
  each citing its Rails `file:line`, each delegating through the instance
  `validatesWith` + `_mergeAttributes` exactly as the two existing ones do.
- `validates_size_of` is an alias of `validates_length_of` on the instance side
  too (`validations/length.rb:127`).
- `pnpm parity:api --package activemodel` does not drop; the instance members
  attribute to their `validations/*.rb` files.
- A `validate do ... end` block can call each helper and have it add errors to the
  record being validated.
