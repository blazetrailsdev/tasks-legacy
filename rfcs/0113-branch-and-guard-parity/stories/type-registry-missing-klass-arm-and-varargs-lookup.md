---
title: "Type::Registry lacks Rails' klass register arm, lookup varargs and error message"
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

Surfaced while porting `type/registry_test.rb` and `type_test.rb` assertions in
PR #6519 (RFC 0105). Both assertion ports are blocked on this deviation.

Rails `ActiveModel::Type::Registry`
(`vendor/rails/activemodel/lib/active_model/type/registry.rb:15-30`):

```ruby
def register(type_name, klass = nil, &block)
  unless block_given?
    block = proc { |_, *args| klass.new(*args) }
    block.ruby2_keywords if block.respond_to?(:ruby2_keywords)
  end
  registrations[type_name] = block
end

def lookup(symbol, ...)
  registration = registrations[symbol]
  if registration
    registration.call(symbol, ...)
  else
    raise ArgumentError, "Unknown type #{symbol.inspect}"
  end
end
```

Three separate gaps in `packages/activemodel/src/type/registry.ts:52-60`:

1. **No `klass` arm.** trails `register(name, factory)` takes only the block
   form. Rails' two-arity `register(:foo, ::String)` has no counterpart, so
   `test_registering_a_new_type` (`test/cases/type_test.rb:16-22`) and
   `"a class can be registered for a symbol"` (`type/registry_test.rb:8-17`)
   cannot be expressed.
2. **No varargs on `lookup`.** trails takes a single `options?` parameter;
   Rails forwards `...` (positional args and kwargs), which
   `registry_test.rb:14-16` pins with `registry.lookup(:bar, 2, :a)` and
   `registry.lookup(:baz, kw: 1)`.
3. **Wrong error message.** trails raises `` `Unknown type: ${name}` ``; Rails
   raises `Unknown type :foo` (`registry.rb:28`, `symbol.inspect`).
   `registry_test.rb:36-44` asserts the message. AR's
   `packages/activerecord/src/type/adapter-specific-registry.ts:185` already
   uses the Rails message, so ActiveModel's is the outlier.

Per the repo Symbol convention a Ruby Symbol value is a colon-prefixed string,
so the converged message is `Unknown type :${name}` when `name` already carries
its colon, matching how the AR sibling spells it.

## Converged shape

`register(typeName, klass?, block?)` with Rails' `klass`-to-block defaulting;
`lookup(symbol, ...args)` forwarding positionals; the Rails `ArgumentError`
message. Then port the two blocked test files' assertions.

The error-message change touches
`packages/activemodel/src/type/registry.test.ts:19,43,50-52`, which currently
assert the trails spelling — update those, they are not Rails test names.

## Acceptance criteria

- `type/registry_test.rb` and `type_test.rb` report 0 assertion-count / -kind /
  -value mismatches in `pnpm parity:test -- --assertions --package activemodel`,
  and the mark is lowered by that amount.
- The `ArgumentError` message matches `Unknown type :foo`, consistent with
  `adapter-specific-registry.ts:185`.
- `pnpm parity:api:calls` / `:args` stay green; no new baseline rows.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0113-branch-and-guard-parity`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
