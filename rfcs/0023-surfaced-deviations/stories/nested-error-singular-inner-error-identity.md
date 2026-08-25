---
title: "Singular validates_associated wraps duplicated inner error (Rails wraps the actual instance)"
status: ready
updated: 2026-07-27
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
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

Surfaced by PR #5091 (nested-error.test.ts, "no index when singular
association"). Rails' `validates_associated` + nested-error path wraps the
child's _actual_ error object: `NestedError.new(association, inner_error)`
where `inner_error` comes straight from `record.errors.objects`
(vendor/rails/activerecord/lib/active_record/associations/nested_error.rb;
errors merged in autosave_association.rb `association_valid?` /
validations/associated.rb). So `error.inner_error.equal?(pet.errors.objects.first)`
holds — identity, not just `==`.

Trails' associated-validation path wraps a _duplicate_ of the inner error
(same base/attribute/type/options, different instance), so the test's
`expect(error.innerError).toBe(pet.errors.objects[0])` fails and PR #5091 had
to downgrade that one assertion to `toStrictEqual` (justified at the call
site, packages/activerecord/src/associations/nested-error.test.ts). The
collection (`has_many`) path preserves identity — only the singular
(`validatesAssociated` on `has_one`) path duplicates. Likely culprit: an
errors copy/`dupWithBase` in the validate-associated or errors-merge path
(packages/activerecord/src/validations/associated.ts /
autosave-association.ts) cloning the child error before wrapping.

## Acceptance criteria

- Singular associated-validation wraps the child's actual error instance,
  matching Rails identity semantics.
- The `toStrictEqual` accommodation in nested-error.test.ts is reverted to
  `toBe` and the deviation comment deleted.
- Collection path unchanged (its identity assertions already pass).
