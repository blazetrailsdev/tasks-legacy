---
title: "activemodel-attribute-methods-missing-code-generator-layer"
status: draft
updated: 2026-08-12
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activemodel"
deps: []
deps-rfc: []
est-loc: 350
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced by RFC 0096 wave 2 (`naming-burndown-2-arel-activemodel`). Three
`naming` call-argument rows in `packages/activemodel/src/attribute-methods.ts`
report `RB [ref:codeGenerator, ...]` / `RB [ref:owner, ...]` against
`TS [ref:host, ...]` / `TS [ref:this, ...]`. These are **not** identifier
renames: `host` is genuinely the class, not a code generator. The rows stand
because trails has no `ActiveSupport::CodeGenerator` layer at all (a3).

`vendor/rails/activemodel/lib/active_model/attribute_methods.rb:211-236`:

```ruby
def eagerly_generate_alias_attribute_methods(new_name, old_name) # :nodoc:
  ActiveSupport::CodeGenerator.batch(generated_attribute_methods, __FILE__, __LINE__) do |code_generator|
    generate_alias_attribute_methods(code_generator, new_name, old_name)
  end
end

def generate_alias_attribute_methods(code_generator, new_name, old_name) # :nodoc:
  ActiveSupport::CodeGenerator.batch(code_generator, __FILE__, __LINE__) do |owner|
    attribute_method_patterns.each do |pattern|
      alias_attribute_method_definition(code_generator, pattern, new_name, old_name)
    end
    attribute_method_patterns_cache.clear
  end
end

def alias_attribute_method_definition(code_generator, pattern, new_name, old_name) # :nodoc:
  ...
  define_call(code_generator, method_name, target_name, mangled_name, parameters, call_args, namespace: :alias_attribute, as: method_name)
end
```

and attribute_methods.rb:272-281 (`define_attribute_methods`), whose
`generate_alias_attribute_methods owner, aliased_name, attr_name` is the third
row.

trails (`packages/activemodel/src/attribute-methods.ts:286-308`) threads
`host: AttributeMethodHost` — the class itself — through all three, because
`generatedAttributeMethods.call(host)` is the direct equivalent of Rails'
generated-methods module and there is no batching generator object in between.

Related pre-existing `shape` rows in the same cluster (already baselined, same
root cause): `defineAttributeMethods -> define_attribute_method`
(`RB [ref:attrName, kwargs{_owner=ref:owner}]` vs `TS [ref:this, ref:attrName]`)
and `defineAttributeMethod -> define_attribute_method_pattern`
(`RB [..., kwargs{as, owner}]` vs `TS [ref:host, ref:pattern, ref:attrName]`).

Renaming `host` to `codeGenerator` would be a lie about what the value is, so
the burndown story deliberately left all three standing.

## Acceptance criteria

- [ ] Decide, with a citation, whether `ActiveSupport::CodeGenerator` is in
      scope for trails: either port it and thread a real `codeGenerator` /
      `owner` through this cluster, or record the decision and the naming
      consequence at the call sites (`@missingRailsCall` / a reviewed baseline
      `reason`) so a future burndown does not re-derive this.
- [ ] If ported: the three `naming` rows and the two related `shape` rows all
      clear `pnpm parity:api:calls:args:report` / `pnpm parity:api:calls:args`.
- [ ] `pnpm vitest run packages/activemodel/src` and the AR attribute-methods /
      alias-attribute tests pass.
