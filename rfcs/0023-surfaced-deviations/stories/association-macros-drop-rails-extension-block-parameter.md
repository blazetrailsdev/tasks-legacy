---
title: "Association macros drop Rails' &extension block parameter, so the block form of an association extension has no spelling"
status: draft
updated: 2026-08-23
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
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

Every Rails association macro takes a trailing block and forwards it to the
builder as the association's extension module:

```ruby
def has_many(name, scope = nil, **options, &extension)
  reflection = Builder::HasMany.build(self, name, scope, options, &extension)
  Reflection.add_reflection self, name, reflection
end
```

(`vendor/rails/activerecord/lib/active_record/associations.rb:1302-1305`;
`has_one` :1498-1500, `belongs_to` :1689-1691,
`has_and_belongs_to_many` :1870 and :1904.)

The block reaches `Builder::Association.build` → `define_extensions`
(`associations/builder/association.rb:44-56`), which wraps it in an anonymous
`Module.new(&extension)` named `<Model>::<Assoc>AssociationExtension` and
appends it to `options[:extension]`. It is how a Rails user writes:

```ruby
has_many :things do
  def find_by_name(name) = where(name: name).first
end
```

None of trails' macros declare the parameter. `Model.hasMany`
(`packages/activerecord/src/associations.ts:765`), `hasOne`, `belongsTo` and
`hasAndBelongsToMany` (:796) take only `(name, scope, options)`, so the block
form has no spelling at all. Extensions are reachable only through the
`extend:` option, which is Rails' _other_ entry point (`:extend` in
`valid_options`), not this one. `CollectionAssociation.defineExtensions`
exists on the TS side but nothing ever passes it a block.

Surfaced while porting the HABTM macro body in #6900: Rails'
`has_many name, scope, **hm_options, &extension` forwards the HABTM
declaration's own block through to the generated through-association, and the
trails line that replaced it can only pass `(name, positionalScope, hmOptions)`.

## Acceptance criteria

- [ ] `hasMany` / `hasOne` / `belongsTo` / `hasAndBelongsToMany` accept Rails'
      trailing block parameter and forward it to `Builder::Association.build`,
      keeping the Rails name `extension`.
- [ ] `Builder::Association.build` passes it to `defineExtensions`, which names
      the generated module `<Model>::<Assoc>AssociationExtension` as
      `associations/builder/association.rb:47-55` does.
- [ ] `hasAndBelongsToMany` forwards its own block to the generated
      through-`hasMany` (`associations.rb:1904`).
- [ ] A test covers the block form declaring a method reachable on the
      collection proxy, mirroring Rails' `associations/extension_test.rb`.
- [ ] `pnpm parity:api:calls` / `:args` green with no new rows.
