---
title: "Port ActiveRecord's alias_attribute override and wire it onto Base"
status: ready
updated: 2026-08-23
rfc: "0115-activemodel-fidelity-convergence"
cluster: null
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

# Port ActiveRecord's `alias_attribute` override and wire it onto `Base`

## Context

Found while landing trails#6940 (`call-define-attribute-methods-from-init-internals`),
which fixed the identical defect one method over: ActiveRecord's
`undefine_attribute_methods` override existed in `attribute-methods.ts` but was
never wired onto `Base`, so `Base` inherited ActiveModel's and neither
generation flag was ever cleared.

`alias_attribute` has the same two problems and they are still live.

Rails, `activerecord/lib/active_record/attribute_methods.rb:66-74`:

```ruby
def alias_attribute(new_name, old_name)
  super

  if @alias_attributes_mass_generated
    ActiveSupport::CodeGenerator.batch(generated_attribute_methods, __FILE__, __LINE__) do |code_generator|
      generate_alias_attribute_methods(code_generator, new_name, old_name)
    end
  end
end
```

The `super` is ActiveModel's (`activemodel/lib/active_model/attribute_methods.rb:203-209`),
which merges `attribute_aliases`, appends to `aliases_by_attribute_name`, and
calls `eagerly_generate_alias_attribute_methods` — which ActiveRecord
deliberately no-ops at `:76-78` ("alias attributes in Active Record are lazily
generated"). The `@alias_attributes_mass_generated` arm is what re-generates an
alias declared AFTER the class's mass generation already ran.

trails, `packages/activerecord/src/attribute-methods.ts:349`:

```ts
export function aliasAttribute(this: AttributeMethodsHost, newName: string, oldName: string): void {
  const amFn = Object.getPrototypeOf(this)?.aliasAttribute;
  if (typeof amFn === "function") {
    amFn.call(this, newName, oldName);
  } else {
    if (!this.attributeAliases) this.attributeAliases = {};
    this.attributeAliases[newName] = oldName;
  }
}
```

Three divergences:

1. **The `@alias_attributes_mass_generated` arm is missing entirely.** An alias
   added after `generateAliasAttributes` has run gets no methods generated.
2. **The `else` branch has no Rails counterpart.** It writes
   `attributeAliases[newName]` directly, skipping `aliasesByAttributeName` and
   `eagerlyGenerateAliasAttributeMethods` — so `generateAliasAttributes`
   (`attribute-methods.ts:520`, which iterates `aliasesByAttributeName`) never
   sees the alias.
3. **It is not wired onto `Base`.** `grep aliasAttribute packages/activerecord/src/base.ts`
   returns nothing, so `Base` inherits ActiveModel's and this function is dead
   code — the same defect #6940 fixed for `undefineAttributeMethods`.

The `Object.getPrototypeOf(this)?.aliasAttribute` "super" is also unsound once
wired: for a subclass, `Object.getPrototypeOf(SubClass)` is `Base`, whose static
would be this same function — infinite recursion. #6940's
`undefineAttributeMethods` shows the working shape: import ActiveModel's
implementation directly under an `am`-prefixed alias and call it, the way
`initializeGeneratedModules` already imports `_coreInitializeGeneratedModules`.

## Converged shape

```ts
export function aliasAttribute(this: AttributeMethodsHost, newName: string, oldName: string): void {
  amAliasAttribute.call(this as never, newName, oldName);

  if (
    Object.prototype.hasOwnProperty.call(this, "_aliasAttributesMassGenerated") &&
    this._aliasAttributesMassGenerated
  ) {
    CodeGenerator.batch(this.generatedAttributeMethods(), __FILE__, __LINE__, (codeGenerator) => {
      generateAliasAttributeMethods.call(this, codeGenerator, newName, oldName);
    });
  }
}
```

with `aliasAttribute: _aliasAttribute` added to the `extend(Base, { ... })`
block at `packages/activerecord/src/base.ts:4722` and a matching
`declare static`, exactly as #6940 did for `undefineAttributeMethods`. The
own-property check on the flag is the port of Rails' per-class ivar (an
inherited `true` belongs to the parent), matching the convention already used
throughout this file.

Check whether ActiveRecord's `eagerly_generate_alias_attribute_methods` no-op
override (`attribute_methods.rb:76-78`) is present and wired too — trails
exports `eagerlyGenerateAliasAttributeMethods` from `attribute-methods.ts` and
does wire it, so confirm it is the no-op arm and not ActiveModel's generating
one.

## Acceptance criteria

- [ ] `aliasAttribute` calls ActiveModel's implementation directly (no
      prototype-chain "super"), and the non-Rails `else` fallback is deleted.
- [ ] The `@alias_attributes_mass_generated` re-generation arm is ported.
- [ ] `aliasAttribute` is wired onto `Base` so ActiveRecord's override is the
      one that runs, with a regression test that fails without the wiring.
- [ ] An alias declared after mass generation gets its methods generated.
- [ ] `attribute-methods.test.ts`, `attribute-methods.trails.test.ts` and
      `base.test.ts` stay green on SQLite, PostgreSQL and MySQL/MariaDB.
