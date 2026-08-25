---
title: "Settle the kwargs-split shape for the *ForBang schema lookups (Ruby key:, **options)"
status: ready
updated: 2026-08-25
rfc: "0119-connection-adapter-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 80
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while landing `check-constraint-raise-message-uses-json-stringify-not-ruby-hash-inspect`
(PR #7046).

Rails splits `check_constraint_for!`'s kwargs in the SIGNATURE
(`vendor/rails/activerecord/lib/active_record/connection_adapters/abstract/schema_statements.rb:1802-1806`):

```ruby
def check_constraint_for!(table_name, expression: nil, **options)
  check_constraint_for(table_name, expression: expression, **options) ||
    raise(ArgumentError, "Table '#{table_name}' has no check constraint for #{expression || options}")
end
```

trails takes one `kwargs` bag and splits on the first body line
(`packages/activerecord/src/connection-adapters/abstract/schema-statements.ts`,
`checkConstraintForBang`):

```ts
async checkConstraintForBang(tableName: string, kwargs: {...} = {}) {
  const { expression, ...options } = kwargs;
```

The direct mirror — a destructured parameter — was tried and reverted: it must
also be restated on the `AbstractAdapter` interface declaration
(`packages/activerecord/src/connection-adapters/abstract-adapter.ts`), which the
`scripts/mixin-declaration-drift.test.ts` guard compares as written text, and TS
rejects destructuring an OPTIONAL parameter in a declaration signature
(`Property 'expression' does not exist on type '{...} | undefined'`). That broke
the build on nine lanes at once (PR #7046, commit e49f49db3, reverted by
9a11270b2).

So this is a genuine TS shortcoming, justified at the call site — but the
sibling `*ForBang` lookups (`foreignKeyForBang`, PG's `uniqueConstraintForBang`
/ `exclusionConstraintForBang`) each take Rails' kwargs differently, and no one
has decided the repo-wide shape.

## Acceptance criteria

- [ ] Settle ONE trails idiom for a Ruby `key: default, **rest` signature on a
      method that is also declared on an adapter interface, and apply it to the
      four `*ForBang` lookups so they read alike.
- [ ] Whatever the shape, the remainder keeps Rails' `options` name (it is what
      the raise message interpolates) and the interface declaration matches the
      implementation textually — verify with
      `pnpm vitest run scripts/mixin-declaration-drift.test.ts`.
- [ ] Verify with `pnpm test:types` (or `tsc --build --force` after deleting
      `packages/*/tsconfig.tsbuildinfo`): the pre-commit `tsc --build` is
      incremental and SKIPS the project, and `pnpm typecheck` passed with the
      broken declaration in place.
- [ ] If the destructured-parameter mirror turns out to be reachable (e.g. by
      making the parameter non-optional with a default on the implementation
      side only), prefer it — it is the closer mirror.
