---
title: "Make abstract adapter #explain a required NotImplementedError member, not an optional type slot"
status: draft
updated: 2026-08-16
rfc: "0119-connection-adapter-fidelity"
cluster: null
packages: []
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
