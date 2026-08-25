---
title: "extended-deterministic-queries-receiver-as-parameter"
status: draft
updated: 2026-08-12
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 70
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while burning down RFC 0096 wave-2 naming rows (PR #6433).

`ExtendedDeterministicQueries::RelationQueries` and `::CoreQueries`
(`vendor/rails/activerecord/lib/active_record/encryption/extended_deterministic_queries.rb`,
`module RelationQueries` / `module CoreQueries`) are Ruby `prepend`/`include`
modules whose methods call `super` and pass `self`:

```ruby
def where(*args)
  super(*EncryptedQuery.process_arguments(self, args, true))
end
```

trails (`packages/activerecord/src/encryption/extended-deterministic-queries.ts:205-265`)
models this as static methods taking the wrapped function _and_ the receiver as
ordinary leading parameters — `RelationQueries.where(originalWhere, relation, args)`,
`CoreQueries.findBy(originalFindBy, klass, args)` — so `self` is spelled
`relation` / `klass`. That is what makes the three RFC 0096 rows
(`process_arguments`: Ruby `ref:this, ref:args, bool:true` → TS
`ref:relation, ref:args, bool:true` and `ref:klass, …`) a3 findings rather than
renames.

Converged shape: the settled trails idiom for Ruby `include`/`prepend` — a
`this`-typed function assigned to the class (CLAUDE.md "Module mixins"), so the
receiver is `this` and only the super-call needs an explicit binding. Note this
cluster is `prepend` with `super`, not plain `include`, so the super handle is
the part that genuinely needs a shape; the receiver does not.

## Acceptance criteria

- [ ] `RelationQueries.where` / `.isExists` / `.scopeForCreate` and
      `CoreQueries.findBy` take the receiver as `this`, not as a positional
      parameter, and pass `this` to `EncryptedQuery.processArguments`.
- [ ] The `super` handle keeps whatever shape is needed, justified at the call
      site if it is not the settled mixin idiom.
- [ ] `pnpm parity:api:calls:args` stays green and the three
      `extended-deterministic-queries.ts` naming rows are gone.
- [ ] The encryption extended-query suites pass unchanged.
