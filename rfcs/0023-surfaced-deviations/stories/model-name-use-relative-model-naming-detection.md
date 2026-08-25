---
title: "Model.modelName detects use_relative_model_naming? like Rails"
status: draft
updated: 2026-08-15
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activemodel"
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

`ActiveModel::Naming#model_name` (naming.rb:271-276) picks the namespace it
passes to `Name.new` by walking the class's module parents:

    namespace = module_parents.detect do |n|
      n.respond_to?(:use_relative_model_naming?) && n.use_relative_model_naming?
    end
    ActiveModel::Name.new(self, namespace)

That argument is the only thing that selects the prefix-dropped
`param_key` / `route_key` shape (naming.rb:171, :180-182).

PR #6572 taught `ModelName` the distinction: a namespace passed as
`{ name, useRelativeModelNaming: true }` gets the isolated shape, a string or
segment array gets the shared one. What is still missing is the _detection_ —
`Model.modelName` (`packages/activemodel/src/model.ts:1531-1537`) and
`Serializers::JSON`'s copy (`packages/activemodel/src/serializers/json.ts:144-150`)
always build the namespace from `this.moduleName?.split("::")`, i.e. always a
plain string array. So no model class in trails can produce the isolated shape;
it is reachable only by constructing a `ModelName` by hand in a test.

This is latent rather than wrong today — trails has no engine /
`isolate_namespace` concept, and Rails' own default for a plain namespaced
class is the shared shape, which is what `Model.modelName` now produces. It
becomes wrong the moment trails grows engines.

## Converged shape

`Model.modelName` mirrors naming.rb:271-276: consult the enclosing namespace
for `use_relative_model_naming?` (Rails' spelling, `useRelativeModelNaming` per
docs/ruby-ts-conventions.md) and pass the object form of the namespace when it
answers truthily, the segment array otherwise. `Serializers::JSON.modelName`
does the same — the two are copies of one Rails method and must not drift.

## Acceptance criteria

- A model whose namespace declares relative model naming gets
  `paramKey` / `routeKey` without the prefix; one that does not keeps it.
- `Model.modelName` and `Serializers::JSON.modelName` derive the namespace
  identically.
- No new rows in any parity baseline.

## Absorbed: `model-module-name-carrier-needs-receipt`

Merged in during the RFC 0023 triage pass (2026-08-18). Original title: "Model.moduleName carries no @noRailsEquivalent receipt"

### Context

Surfaced in PR #6568. `packages/activemodel/src/model.ts`'s
`declare static moduleName?: string` is the `::`-joined module-path carrier
that `model_name` passes as Rails' `namespace` argument
(`activemodel/lib/active_model/naming.rb:271-275`, where Ruby gets it from
`module_parents` on the constant itself).

It scores as **novel** extra surface in
`pnpm parity:api:extra --package activemodel` with no tag. The identical
carrier on `packages/activemodel/src/serializers/json.ts:63` was tagged
`@noRailsEquivalent PERMANENT` in that PR — the `model.ts` twin predates it and
was left untouched to keep the diff scoped.

Note this is a receipt, not a convergence: a JS class name genuinely carries no
module path, which is the language-level fact the PERMANENT classification
records. If the carrier can be eliminated (e.g. by deriving the path from a
registry at `model_name` time) that is the better outcome and this story should
take it.

### Acceptance criteria

- [ ] `model.ts`'s `moduleName` either stops being extra surface, or carries a
      `@noRailsEquivalent PERMANENT` reason citing naming.rb:271-275.
- [ ] `pnpm parity:api:extra`'s permanence census stays at 0 unclassified.
