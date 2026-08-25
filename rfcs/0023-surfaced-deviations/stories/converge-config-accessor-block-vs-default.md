---
title: "config_accessor conflates Rails' default: kwarg with its block (a function default is invoked, not stored)"
status: draft
updated: 2026-08-17
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activesupport"
deps: []
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

Rails' `config_accessor` takes a `default:` kwarg AND a block, and they are
distinct inputs:

```ruby
# vendor/rails/activesupport/lib/active_support/configurable.rb:107,126
def config_accessor(*names, instance_reader: true, instance_writer: true, instance_accessor: true, default: nil)
  ...
  send("#{name}=", block_given? ? yield : default)
```

The block, when given, is _called_ and its result stored; `default:` is stored
verbatim. `configurable_test.rb:53-62` exercises the block form
(`config_accessor :hair_colors, :tshirt_colors do [:black, :blue, :white] end`)
and `:64-71` the kwarg form (`config_accessor :foo, default: :bar`).

trails collapses the two: `packages/activesupport/src/configurable.ts` stores

```ts
this[name] = typeof options.default === "function" ? options.default() : options.default;
```

so a function reaching `default:` is invoked instead of stored. A Ruby Proc
passed as `default:` — legal, and stored as-is by Rails — would be called
and its return value stored instead. `configurable.test.ts`'s
`configuration accessors can take a default value as a block` therefore spells
Rails' block as `{ default: () => [...] }`.

This mirrors the same conflation in `mattrAccessor`
(`packages/activesupport/src/module-ext.ts`, `MattrOptions`), which is where
the shape was copied from — so converging one should settle the idiom for both.
Per CLAUDE.md, an existing deviation next to the work is a reason to converge
it, not to match it; this story records the debt PR #6654 propagated rather
than invented.

## Converged shape

A trailing block parameter distinct from `default:`, which is the settled
trails idiom for a Ruby block (`config_accessor(names..., options, block?)`),
so `default:` stores any value verbatim — function included — and only the
block is invoked. Rails' `block_given? ? yield : default` ordering (block wins
over `default:`) must be preserved.

Note the interaction with the rest-args shape: `configAccessor` is
`(...namesAndOptions)` with the options object popped off the end, so a block
needs a position that cannot be confused with a name or the options hash.

## Acceptance criteria

- `configAccessor(klass, "foo", { default: someFunction })` stores
  `someFunction` itself, not its return value.
- The block form still evaluates once per call and wins over `default:`.
- `configuration accessors can take a default value as a block` /
  `... as an option` in `configurable.test.ts` both use the converged shapes;
  names unchanged; configurable_test.rb stays 0/0/0 on assertions.
- Decide and record whether `mattrAccessor`/`mattrReader` converge in the same
  PR or in a follow-up filed against this RFC.
