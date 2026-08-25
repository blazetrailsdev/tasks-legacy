---
title: "CheckConstraintDefinition#defined_for? ports the fetch(:validate) and **options arms"
status: in-progress
updated: 2026-08-25
rfc: "0051-migration-schema-statements-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 60
priority: 14
pr: 7017
claim: "2026-08-24T23:54:09Z"
assignee: "move-ts-only-extras-out-of-mirrored-activemodel-attribute-test-file"
blocked-by: null
closed-reason: null
---

## Context

Surfaced while converging `CheckConstraintDefinition`'s constructor to Rails'
Struct shape in PR #6360 (`call-args-ar-kwargs-vs-positional`). The ctor,
`name`, and `validate?` now mirror
`activerecord/lib/active_record/connection_adapters/abstract/schema_definitions.rb:175-183`,
but `defined_for?` was left alone.

Rails (`schema_definitions.rb:189-195`):

```ruby
def defined_for?(name:, expression: nil, validate: nil, **options)
  options = options.slice(*self.options.keys)

  self.name == name.to_s &&
    (validate.nil? || validate == self.options.fetch(:validate, validate)) &&
    options.all? { |k, v| self.options[k].to_s == v.to_s }
end
```

trails (`schema-definitions.ts`, `isDefinedFor`) compares against the derived
`this.validate` getter and ignores the `**options` arm entirely:

```ts
return (
  this.name === (options.name == null ? "" : options.name.toString()) &&
  (options.validate === undefined || options.validate === this.validate)
);
```

Two divergences:

- `self.options.fetch(:validate, validate)` returns the _stored_ value whenever
  the key exists — including a stored `false` — and otherwise echoes the
  caller's own `validate`, so an unset `:validate` always matches. Comparing
  against `this.validate` (which defaults to `true`) makes
  `defined_for?(validate: false)` fail against a constraint that never set the
  key. This is the `fetch`-vs-`??` trap in CLAUDE.md.
- The `**options` residual arm — a stringified compare over the remaining keys,
  narrowed by `slice(*self.options.keys)` — is absent.

Now that `options` is a real member on the TS class, both arms are portable.

## Acceptance criteria

1. `isDefinedFor` mirrors `schema_definitions.rb:189-195` branch for branch,
   including the `fetch(:validate, validate)` semantics and the `**options`
   residual compare.
2. A test covers `defined_for?(validate: false)` against a constraint with no
   stored `:validate`, and fails on the current implementation.
3. Existing `remove_check_constraint` / introspection call sites still pass on
   all three adapters.
