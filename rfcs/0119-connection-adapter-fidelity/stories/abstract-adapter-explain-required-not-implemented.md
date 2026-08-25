---
title: "Make abstract adapter #explain a required NotImplementedError member, not an optional type slot"
status: blocked
updated: 2026-08-25
rfc: "0119-connection-adapter-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 120
priority: null
pr: 7041
claim: "2026-08-25T15:30:56Z"
assignee: "port-attribute-method-pattern-match-struct"
blocked-by: "Blocked on schema-statements-host-type-inherits-adapter-surface: interface SchemaStatements extends DatabaseAdapter (abstract/schema-statements.ts:313) republishes every REQUIRED adapter member as extra surface, so flipping explain? to explain moves activerecord 1392->1393 and reds parity:api:extra:gate. The NotImplementedError base body landed in #7041; the required-member half waits on that inheritance being removed."
closed-reason: null
---

## Context

`ActiveRecord::ConnectionAdapters::DatabaseStatements#explain`
(`connection_adapters/abstract/database_statements.rb:180-182`) is a REQUIRED
member of the abstract adapter that raises when unimplemented:

```ruby
def explain(arel, binds = [], options = []) # :nodoc:
  raise NotImplementedError
end
```

trails declares it OPTIONAL instead —
`packages/activerecord/src/connection-adapters/abstract-adapter.ts:779`:

```ts
explain?(arel: unknown, binds?: unknown[], options?: ExplainOption[]): Promise<string>;
```

There is no `NotImplementedError`-raising base body, so the abstract adapter
does not declare the member at all in the Rails sense; it is a `?:` on the type.

That divergence leaks into ported call sites. `Explain#exec_explain`
(`explain.rb:24`) calls `c.explain(sql, binds, options)` unconditionally, so the
trails port in `packages/activerecord/src/explain.ts` has to spell it
`c.explain!(...)` with a non-null assertion and a comment explaining that every
adapter reaching that point implements it. Surfaced by PR #6598
(`converge-explain-exec-and-build-clause-into-one-mixin`), which replaced
`relation.ts`'s invented `"EXPLAIN not supported by this adapter"` early-return
with Rails' unconditional call.

## Converged shape

`explain` becomes a required member on the abstract adapter with a body of
`raise NotImplementedError` (`database_statements.rb:180-182`), the way every
other abstract-adapter stub in trails spells that. Concrete adapters already
override it, so no adapter behaviour changes; what changes is that an adapter
which does NOT implement it fails loudly at the Rails message and site instead
of being silently typed away.

The `!` assertion and its justifying comment in `explain.ts`'s `execExplain`
are then deleted, leaving the call as Rails writes it.

## Acceptance criteria

- [ ] `explain` is a required (non-optional) member on the abstract adapter,
      with a `NotImplementedError`-raising base body matching
      `database_statements.rb:180-182`.
- [ ] `explain.ts`'s `execExplain` calls `c.explain(sql, binds, options)` with
      no non-null assertion and no accompanying justification comment.
- [ ] No adapter gains or loses EXPLAIN behaviour; `itIfSupports("explain", …)`
      guards are unchanged.
- [ ] `pnpm parity:api` delta non-negative; `pnpm parity:api:calls` / `:args`
      clean.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.

## Progress (PR #7041, 2026-08-25)

The base body landed: `explain` in
`connection-adapters/abstract/database-statements.ts` now raises
`NotImplementedError` with Rails' `(arel, binds = [], options = [])` signature
(`database_statements.rb:180-182`), replacing the invented
`throw new Error("explain must be implemented by adapter subclass")`.

The remaining half — making the member REQUIRED on the abstract adapter and
dropping `execExplain`'s `!` — is blocked on
`schema-statements-host-type-inherits-adapter-surface`. Measured on that PR:
`interface SchemaStatements extends DatabaseAdapter`
(`abstract/schema-statements.ts:313`) republishes every REQUIRED adapter member
as public surface of `schema-statements.ts`, so flipping `explain?` to `explain`
adds the extra-surface row `schema-statements.ts::explain` and moves
activerecord 1392 -> 1393, reddening `pnpm parity:api:extra:gate`. Optional
members are exempt from that count; reverting only the `?` restored 1392. The
mark must not be raised, so the inheritance has to go first.
