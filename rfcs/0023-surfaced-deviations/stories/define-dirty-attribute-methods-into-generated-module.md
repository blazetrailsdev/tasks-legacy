---
title: "Generate the dirty attribute-method cascade into generated_attribute_methods"
status: draft
updated: 2026-08-12
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activemodel"
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

Rails generates every per-attribute dirty method into the module
`generated_attribute_methods` builds and includes, via the
`attribute_method_suffix` / `attribute_method_affix` declarations in
`vendor/rails/activemodel/lib/active_model/dirty.rb:141-149` — so
`undefine_attribute_methods`
(`vendor/rails/activemodel/lib/active_model/attribute_methods.rb:296-301`,
`generated_attribute_methods.module_eval { undef_method(*instance_methods) }`)
clears them, and a class-body method outranks them.

trails' `defineDirtyAttributeMethods`
(`packages/activemodel/src/attribute-methods.ts:600-`) takes a `prototype`
argument and installs the whole cascade (`nameChanged`, `nameWas`,
`savedChangeToName`, `restoreName`, …) straight onto the class prototype. Its
callers are `packages/activemodel/src/attributes.ts:217`,
`packages/activemodel/src/attribute-methods.ts` (`aliasAttribute`) and
`packages/activerecord/src/model-schema.ts:1195`.

Since PR #6389 the module is real and in the prototype chain, so this is now the
remaining set of generated methods that live outside it: they survive
`undefineAttributeMethods`, and they shadow rather than defer to a class-body
method of the same name.

## Converged shape

Define the cascade into `generatedAttributeMethods.call(host)` — via
`defineMethod` / `moduleEval`, as the other generation sites now do — rather
than onto `host.prototype`, dropping the `prototype` parameter in favour of the
host so each caller passes the class.

## Acceptance criteria

- `defineDirtyAttributeMethods` installs into the generated-attribute-methods
  module; no generated dirty method is an own property of the class prototype.
- `undefineAttributeMethods` removes the dirty cascade along with the rest.
- A class-body `nameChanged()` outranks the generated one (Rails ancestry).
- The three call sites above are updated; dirty and attribute-method suites stay
  green on all three adapters.
