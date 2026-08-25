---
title: "Type.lookup drops Rails' positional constructor args"
status: draft
updated: 2026-08-19
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 60
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`ActiveRecord::Type.lookup` forwards **positional** args through to the
registration, which passes them to the type's constructor:

- `activerecord/lib/active_record/type.rb:41-43` —
  `def lookup(*args, adapter: current_adapter_name, **kwargs)` →
  `registry.lookup(*args, adapter: adapter, **kwargs)`
- `activerecord/lib/active_record/type/adapter_specific_registry.rb:27-35` —
  `def lookup(symbol, *args, **kwargs)` → `registration.call(self, symbol, *args, **kwargs)`

trails' `lookup` (`packages/activerecord/src/type.ts:136-142`) takes only
`(symbol: string, options?: { adapter?: string; [key: string]: unknown })` —
there is no positional-args arm at all, so a caller cannot reach the
constructor-argument path.

The gap is visible in the port of `type_test.rb`. Rails'
`test "registering a new type"` (`activerecord/test/cases/type_test.rb:15-20`)
is:

```ruby
type = Struct.new(:args)
ActiveRecord::Type.register(:foo, type)
assert_equal type.new(:arg), ActiveRecord::Type.lookup(:foo, :arg)
```

`packages/activerecord/src/type.test.ts` can only assert
`expect(lookup("foo")).toStrictEqual(new ArgType())` — the `:arg` the Rails
test threads through `lookup` into `type.new(:arg)` is dropped, so the
argument-forwarding behaviour is untested and unported. (PR #6736 converged
the assertion _kinds_ for this file; the missing positional arm is what stops
the assertion from being value-identical to Rails.)

## Acceptance criteria

- `lookup` accepts Rails' positional args ahead of the options/kwargs, mirroring
  `type.rb:41` and `adapter_specific_registry.rb:27`, and forwards them to the
  registration/constructor.
- `type.test.ts`'s `registering a new type` asserts what
  `type_test.rb:19` asserts, with `:arg` threaded through: the looked-up type
  equals a type constructed with that argument.
- `pnpm parity:api:calls` / `pnpm parity:api:calls:args` stay clean; no new
  baseline rows.
