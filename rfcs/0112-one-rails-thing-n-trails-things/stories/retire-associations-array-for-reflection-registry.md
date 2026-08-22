---
title: "Retire model._associations in favour of the reflection registry"
status: ready
updated: 2026-08-22
rfc: "0112-one-rails-thing-n-trails-things"
cluster: split-stores
packages: []
deps: []
deps-rfc: []
est-loc: 400
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

# Retire `model._associations` in favour of the reflection registry

## Context

trails carries two parallel reflection stores. `Builder::Association.createReflection`
(`packages/activerecord/src/associations/builder/association.ts`) does both:

```ts
model._associations.push({ type: macro, name, scope, options: { ...options } });
Reflection.addReflection(model, name, reflection);
```

Rails does only the second — `create_reflection` ends at
`ActiveRecord::Reflection.create(macro, name, scope, options, model)`
(`associations/builder/association.rb:48-49`), and `add_reflection`
(`reflection.rb:18-22`) is the single registry. There is no lightweight
per-model array beside it.

The duplicate exists because trails' loaders take an owner/name/definition
triple rather than an association instance, and `AssociationDefinition`
(`associations.ts`) has grown to shadow the reflection reader-for-reader:
`macro`, `foreignKey`, `associationPrimaryKey`, `counterCacheColumn`,
`hasCachedCounter`, `inverseName`, `klass`, and now `scope` — each documented
against the Rails reader it stands in for. `Association#reflection`
(`associations/association.ts:33`) is typed as this definition, not as the rich
reflection, so the two must be kept in sync by hand at every write site.

Surfaced during PR #6512, which moved the association scope out of the options
hash and onto the definition (mirroring `MacroReflection`'s sibling `@scope` /
`@options`, `reflection.rb:376-392`). That convergence was correct for the
options hash, but it also made the definition-vs-reflection duplication harder
to miss: the same scope is now stored twice, once on each object.

## Converged shape

One store. `_reflectOnAssociation` / `Reflection.addReflection` is the registry
Rails has, so `Association#reflection` should hold the reflection that
`Reflection.create` returned, and `model._associations` should go. The
definition's readers already name their Rails counterparts, so each one either
resolves to the existing reflection method or shows up as a gap in the
reflection port worth filing separately.

Expect this to be several PRs: the loaders (`has-many-association.ts`,
`singular-association.ts`, `has-many-through-association.ts`,
`collection-proxy.ts`, `preloader/**`) all read the definition today, and two
paths build ad-hoc definitions with no registered reflection at all — the
`findCollectionTarget` test helper and `loadHasManyThrough`'s synthesised
`sourceType` scope. Those need a reflection-shaped answer before the array can
be deleted.

## Acceptance criteria

- [ ] `model._associations` is gone; `Reflection.addReflection` is the only
      registry.
- [ ] `Association#reflection` is the object `Reflection.create` returned.
- [ ] No reader is duplicated between `AssociationDefinition` and the
      reflection classes.
- [ ] `pnpm parity:api` / `parity:api:extra` deltas non-negative.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
