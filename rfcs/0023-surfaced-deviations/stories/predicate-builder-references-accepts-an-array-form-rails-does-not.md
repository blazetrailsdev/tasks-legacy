---
title: "PredicateBuilder.references accepts an array form Rails does not have"
status: draft
updated: 2026-08-18
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

Found while converging the `references -> sql` call-set row (PR #6721,
`wave-4a-relation-family-residue`).

Rails (`vendor/rails/activerecord/lib/active_record/relation/predicate_builder.rb:28-36`):

```ruby
def self.references(attributes)
  attributes.each_with_object([]) do |(key, value), result|
    if value.is_a?(Hash)
      result << Arel.sql(key, retryable: true)
    elsif (idx = key.rindex("."))
      result << Arel.sql(key[0, idx], retryable: true)
    end
  end
end
```

One parameter, a Hash, iterated as pairs.

trails (`packages/activerecord/src/relation/predicate-builder.ts`, static
`references`) accepts `string[] | Record<string, unknown>` and adds an
array-form arm with no Rails counterpart:

```ts
const entries: Array<[string, unknown]> = Array.isArray(conditions)
  ? conditions.map((k) => [k, undefined] as [string, unknown])
  : Object.entries(conditions);
```

Rails' only caller is `build_where_clause` (`query_methods.rb:1640`), which
passes the transformed **Hash** `opts`. The array arm is invented surface: a
second accepted shape that Rails' signature does not have, kept alive by trails
callers (if any) rather than by the port.

## Converged shape

Narrow the signature to the Hash form and iterate pairs, matching
predicate_builder.rb:29. Audit callers first — `grep -rn "PredicateBuilder.references"`
— and convert any array-form caller to pass a Hash (or to the correct Rails
call), rather than keeping the arm to serve them.

## Acceptance criteria

- [ ] `PredicateBuilder.references` takes the Hash/attributes form only; the
      `Array.isArray` arm is gone.
- [ ] Every caller passes what Rails passes (query_methods.rb:1640 shape).
- [ ] `pnpm parity:api:extra --package activerecord` does not regress; no
      `@noRailsEquivalent` tag added to preserve the arm.
- [ ] `pnpm parity:api:calls` / `:args` green.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.
