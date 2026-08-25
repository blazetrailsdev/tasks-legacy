---
title: "converge-access-slice-with-indifferent-access"
status: done
updated: 2026-08-25
rfc: "0115-activemodel-fidelity-convergence"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: 7010
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`ActiveModel::Access#slice` returns a `HashWithIndifferentAccess`
(`vendor/rails/activemodel/lib/active_model/access.rb:8`):

```ruby
def slice(*methods)
  methods.flatten.index_with { |method| public_send(method) }.with_indifferent_access
end
```

`packages/activemodel/src/access.ts`'s `slice` returns a plain object, and says
so at the call site. `fan-out-model-serialization-conversion-access-naming-surface`
converged the `public_send` half of both bodies but left the return type, because
it is not a local decision:

- Ruby's `HashWithIndifferentAccess` subclasses `Hash`, so one object answers
  `h[:name]`, `h["name"]`, `h.slice(...)` and `h == { "name" => … }`. Rails'
  own test asserts exactly that — `assert_equal({ "id" => 1, "name" => "bob" },
person.slice(:id, :name))`
  (`vendor/rails/activemodel/test/cases/access_test.rb`).
- trails' `HashWithIndifferentAccess`
  (`packages/activesupport/src/hash-with-indifferent-access.ts:81-88`) is
  `Map`-backed and answers only through `get()` / `set()` — JS has no `Hash` to
  subclass. Returning it flips `sliced.email` and `expect(sliced).toEqual({…})`
  (`packages/activemodel/src/access.test.ts:15-24`, which mirror the Rails
  assertions) from working to not working.

So the question is what a trails method whose Ruby counterpart returns a HWIA
should return, package-wide — `store.ts:588` already returns one from
`withIndifferentAccess`, so both answers exist in the tree today.

## Acceptance criteria

- A decision recorded for HWIA-returning methods: either `slice` returns
  `withIndifferentAccess(...)` and the Rails-mirroring assertions in
  `access.test.ts` are re-spelled against `get()` (test NAMES unchanged), or the
  plain-object return is ratified once, centrally, with the JS `Hash`-subclass
  shortcoming as its reason — and `access.ts`'s call-site note is replaced by a
  pointer to it.
- Whichever way it lands, `access.ts` no longer carries a story pointer.
- `pnpm vitest run packages/activemodel/src/access.test.ts`; parity deltas
  non-negative.
