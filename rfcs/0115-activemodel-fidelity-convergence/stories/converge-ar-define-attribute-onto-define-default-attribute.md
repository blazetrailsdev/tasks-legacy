---
title: "Converge AR define_attribute onto define_default_attribute + Attribute::UserProvidedDefault"
status: in-progress
updated: 2026-08-23
rfc: "0115-activemodel-fidelity-convergence"
cluster: api-compare
packages: ["activerecord", "activemodel"]
deps: ["retire-attribute-definitions-registry-for-default-attributes"]
deps-rfc: []
est-loc: 260
pr: 6961
claim: "2026-08-23T23:16:55Z"
assignee: "converge-ar-define-attribute-onto-define-default-attribute"
blocked-by: null
closed-reason: null
---

## Context

Rails' `ActiveRecord::Attributes::ClassMethods#define_attribute`
(`activerecord/lib/active_record/attributes.rb:231-239`) is two statements:

```ruby
def define_attribute(name, cast_type, default: NO_DEFAULT_PROVIDED, user_provided_default: true)
  attribute_types[name] = cast_type
  define_default_attribute(name, default, cast_type, from_user: user_provided_default)
end
```

and `define_default_attribute` (`attributes.rb:277-291`) is where the
`from_user:` fork lives — it builds `ActiveModel::Attribute::UserProvidedDefault`
when true, `ActiveModel::Attribute.from_database` when false, and with
`NO_DEFAULT_PROVIDED` it only re-types the existing attribute:

```ruby
def define_default_attribute(name, value, type, from_user:)
  if value == NO_DEFAULT_PROVIDED
    default_attribute = _default_attributes[name].with_type(type)
  elsif from_user
    default_attribute = ActiveModel::Attribute::UserProvidedDefault.new(
      name, value, type, _default_attributes.fetch(name.to_s) { nil })
  else
    default_attribute = ActiveModel::Attribute.from_database(name, value, type)
  end
  _default_attributes[name] = default_attribute
end
```

trails' `defineAttribute` (`packages/activerecord/src/attributes.ts`) has
neither statement. It writes an `AttributeDefinition` row into
`_attributeDefinitions`, and — since PR #6789 retired the stored
`userProvidedDefault` field — expresses `from_user:` by pushing onto the
pending-modification queue only when the default IS user-provided, so the
later `_defaultAttributes` replay applies `with_user_default` (cast) while a
`from_user: false` definition stays a bare column seed (deserialize). Same
observable split, different mechanism, and no `define_default_attribute` and
no `Attribute::UserProvidedDefault` construction anywhere on the path.

`UserProvidedDefault` itself IS ported
(`packages/activemodel/src/attribute/user-provided-default.ts`) and unused by
this path.

Two knock-on shapes go away with this:

- `packages/activerecord/src/attributes.ts` reaches
  `pushPendingType` / `pushPendingDefault` across the package boundary through
  the `@blazetrails/activemodel/attribute-registration` deep import (added in
  #6789 precisely because Rails keeps the queue private,
  `attribute_registration.rb:17,77`). Rails' `define_attribute` never touches
  the queue, so the import — and its vitest-alias + two dx-test tsconfig
  registrations — disappear once this converges.
- `NO_DEFAULT` (the local `Symbol`) becomes Rails' `NO_DEFAULT_PROVIDED`
  private constant (`attributes.rb:274-275`) read by `define_default_attribute`
  rather than by the caller.

## Converged shape

`defineAttribute` becomes the two Rails statements, and a private
`defineDefaultAttribute(name, value, type, { fromUser })` carries the fork,
building `UserProvidedDefault` / `Attribute.fromDatabase` and writing into
`_defaultAttributes`.

Sequence after
`retire-attribute-definitions-registry-for-default-attributes`: writing into
`_default_attributes` directly is only meaningful once that IS the registry.

## Acceptance criteria

- [ ] `defineAttribute` mirrors attributes.rb:231-239 — two statements, no
      `_attributeDefinitions` write.
- [ ] A private `defineDefaultAttribute` mirrors attributes.rb:277-291,
      including the `NO_DEFAULT_PROVIDED` arm and the
      `_default_attributes.fetch(name) { nil }` original passed to
      `UserProvidedDefault`.
- [ ] `packages/activemodel/src/attribute/user-provided-default.ts` is
      constructed on this path.
- [ ] The `@blazetrails/activemodel/attribute-registration` deep import is gone
      from `packages/activerecord/src/attributes.ts`, along with its
      `vitest.config.ts` alias and both dx-test tsconfig `paths` entries (if no
      other importer remains).
- [ ] `pnpm parity:api:calls` / `:args` add zero rows; parity deltas
      non-negative.
