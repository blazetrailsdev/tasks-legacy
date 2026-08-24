---
title: "ActiveModel _read_attribute carries ActiveRecord's body"
status: claimed
updated: 2026-08-24
rfc: "0115-activemodel-fidelity-convergence"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 260
priority: null
pr: null
claim: "2026-08-24T23:06:10Z"
assignee: "fan-out-model-json-serializer-surface"
blocked-by: null
closed-reason: null
---

## Context

`ActiveModel::AttributeMethods#_read_attribute`
(`vendor/rails/activemodel/lib/active_model/attribute_methods.rb:556-558`) is a
plain dispatch to the reader method:

```ruby
def _read_attribute(attr)
  __send__(attr)
end
```

`ActiveRecord::AttributeMethods::Read#_read_attribute`
(`vendor/rails/activerecord/lib/active_record/attribute_methods/read.rb:35-37`)
is the one that goes to the attribute set:

```ruby
def _read_attribute(attr_name, &block)
  @attributes.fetch_value(attr_name, &block)
end
```

trails' ActiveModel port (`packages/activemodel/src/attribute-methods.ts`,
`_readAttribute`) carries the ActiveRecord body — `_attributes.fetchValue(attr,
block) ?? null` — plus a `block` parameter ActiveModel's signature does not
have. So ActiveModel bypasses the reader method where Rails dispatches through
it, which means a user-defined or aliased reader is not consulted at the
ActiveModel layer.

PR #7000 moved this body out of `model.ts` into `attribute-methods.ts` and
removed a self-recursion guard that only existed because `Model` shadowed the
function, but did not converge the body itself — the AR-shaped read was the
pre-existing behaviour and every caller depends on it.

## Converged shape

ActiveModel's `_read_attribute` should be `__send__(attr)` — in trails, reading
the generated accessor property for `attr` — with ActiveRecord's `read.ts`
override supplying the `@attributes.fetch_value(name, &block)` body and the
`block` parameter. That is the split Rails has, and `read.rb:11` already exists
as the AR-side hook.

Note the coupling to CLAUDE.md § "Generated attribute readers are properties":
`__send__(attr)` on a generated reader is a property read, not a call, so the
converged ActiveModel body reads `(this as Record<string, unknown>)[attr]`.

## Acceptance criteria

- `attribute-methods.ts`'s `_readAttribute` dispatches through the reader, with
  no `block` parameter, matching attribute_methods.rb:556-558.
- ActiveRecord keeps the `fetch_value(name, &block)` body in its own
  `attribute-methods/read.ts`, matching read.rb:35-37.
- The MissingAttributeError path the block feeds (generated getters, `[]`) still
  raises from the ActiveRecord side.
- `pnpm parity:api:calls` / `:args` clean; activemodel and activerecord parity
  deltas non-negative.
