---
title: "A class-body reader makes its attribute unassignable — the generated writer is unreachable"
status: draft
updated: 2026-08-19
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activemodel"
deps: ["bare-pattern-generates-reader-not-accessor-property"]
deps-rfc: []
est-loc: 120
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced in PR #6738 while converging ActiveModel's `_assignAttribute` onto
`attribute_assignment.rb:67-75`. Removing that method's non-Rails
`writeAttribute` fallback made the symptom visible; the fallback had been
masking it.

A class that declares an attribute AND defines its own reader in the class body
loses the generated writer's JS-assignment spelling:

```ts
class Person extends Model {
  static {
    this.attribute("name", "string");
  }
  get name(): string {
    return "OVERRIDE:" + (this.readAttribute("name") as string);
  }
}
```

Verified on that class: `person["name="]` IS a function — the `attribute=`
pattern generated its writer — but no `set` accessor for `name` exists anywhere
on the prototype chain, so `person.name = "x"` reaches the class body's get-only
accessor instead of the writer.

Rails has no such hole. `instance_method_already_implemented?`
(`vendor/rails/activemodel/lib/active_model/attribute_methods.rb:404-406`) is
`generated_attribute_methods.method_defined?(method_name)` — it consults the
generated module ONLY, so `attribute :name` generates both `name` and `name=`
into that module regardless of what the class body defines. The class's own
`def name` then wins for the reader by ancestry (class beats included module),
while `name=` is untouched. Ruby implements "the class wins" through the
ancestor chain; trails implements it by not generating, and a JS accessor
property cannot take its `get` from the class and its `set` from the generated
module (`attributes.rb:92-102`'s note that "a property cannot take its halves
from two").

trails' own `isInstanceMethodAlreadyImplemented`
(`packages/activemodel/src/attribute-methods.ts:520-525`) is a faithful port of
:404-406 — it consults the generated module. The divergence is downstream of the
predicate, in how the reader is installed.

This is the same root as [[bare-pattern-generates-reader-not-accessor-property]]
— readers are accessor properties rather than generated methods — recorded here
as the concrete symptom that root produces, so the sweep has a failing case to
aim at.

## Converged shape

Falls out of `bare-pattern-generates-reader-not-accessor-property`: once the
bare pattern generates a plain reader METHOD into the generated module rather
than an accessor property on the prototype, the class body's `get name()` and
the generated `name=` coexist exactly as Ruby's do, and `person.name = "x"`
resolves to the writer.

If that story lands without covering this case, the check to add is that the
`attribute=` pattern's writer is reachable through JS assignment for a name the
class body also reads.

## Acceptance criteria

- [ ] `person.name = "x"` reaches the generated writer on a class that declares
      `name` and defines `get name()` in its body.
- [ ] A test pins that case; verified failing on the baseline.
- [ ] No test renames.
