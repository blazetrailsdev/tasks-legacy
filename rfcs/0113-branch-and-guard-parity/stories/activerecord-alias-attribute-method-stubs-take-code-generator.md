---
title: "ActiveRecord alias-attribute generation is a stub where Rails loops patterns, clears the cache and raises"
status: ready
updated: 2026-08-25
rfc: "0113-branch-and-guard-parity"
cluster: missing-arm
packages: []
deps: []
deps-rfc: []
est-loc: 220
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`packages/activerecord/src/attribute-methods.ts:305-340` carries stubs where
`activerecord/lib/active_record/attribute_methods.rb:76-96` has real bodies:

```ruby
def eagerly_generate_alias_attribute_methods(_new_name, _old_name) # :nodoc:
  # alias attributes in Active Record are lazily generated
end

def generate_alias_attribute_methods(code_generator, new_name, old_name) # :nodoc:
  attribute_method_patterns.each do |pattern|
    alias_attribute_method_definition(code_generator, pattern, new_name, old_name)
  end
  attribute_method_patterns_cache.clear
end

def alias_attribute_method_definition(code_generator, pattern, new_name, old_name) # :nodoc:
  old_name = old_name.to_s
  if !abstract_class? && !has_attribute?(old_name)
    raise ArgumentError, "#{self.name} model aliases `#{old_name}`, but `#{old_name}` is not an attribute. " \
      "Use `alias_method :#{new_name}, :#{old_name}` or define the method manually."
  else
    define_attribute_method_pattern(pattern, old_name, owner: code_generator, as: new_name, override: true)
  end
end
```

trails' `generateAliasAttributeMethods` is an empty body with a comment
claiming "This hook exists for Rails parity"; `aliasAttributeMethodDefinition`
takes `(newName, oldName)` with no `codeGenerator`/`pattern`, defines a
getter/setter straight onto `this.prototype`, and drops the
`ArgumentError` arm entirely (aliasing a non-attribute silently succeeds where
Rails raises). `ActiveSupport::CodeGenerator` is now ported
(`packages/activesupport/src/code-generator.ts`, PR #6527), so the reason these
were stubs no longer holds.

Note the call-set gate does NOT flag the empty body: `compare.ts#effectiveTsCalls`
resolves through `delegateCalls`, which follows package deps, so the activemodel
function of the same name discharges AR's flags. The divergence is real
regardless.

## Acceptance criteria

- [ ] `generateAliasAttributeMethods(codeGenerator, newName, oldName)` loops
      `attributeMethodPatterns` and clears `attributeMethodPatternsCache`, per
      attribute_methods.rb:80-85.
- [ ] `aliasAttributeMethodDefinition(codeGenerator, pattern, newName, oldName)`
      raises `ArgumentError` with Rails' exact message when the class is not
      abstract and lacks the attribute, and otherwise routes through
      `defineAttributeMethodPattern(pattern, oldName, { owner: codeGenerator, as: newName, override: true })`
      (attribute_methods.rb:87-96).
- [ ] `eagerlyGenerateAliasAttributeMethods(_newName, _oldName)` keeps Rails'
      two-parameter no-op signature (attribute_methods.rb:76-78) rather than
      the current zero-arg flag setter.
- [ ] A test covers the `ArgumentError` arm, matching Rails' name for it.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0113-branch-and-guard-parity`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
