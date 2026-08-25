---
title: "writerMethod extracts a _decompose helper Rails does not have"
status: draft
updated: 2026-08-21
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
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

Surfaced landing PR #6828 (`composed-of-local-derivations`, RFC 0099).

Rails' `writer_method` decomposes inline, twice, with one line each
(`vendor/rails/activerecord/lib/active_record/aggregations.rb:273-278`):

```ruby
mapping.each { |key, value| write_attribute(key, part.send(value)) }
@aggregation_cache[name] = part.dup.freeze
```

trails extracts that into a module-level `_decompose(record, cache, name,
mapping, value)` helper (`packages/activerecord/src/aggregations.ts:143-165`)
which the setter calls from three separate arms. Rails has no such method, so
this is extra decomposition surface — CLAUDE.md "No extra abstraction" and
"Decomposition: one Rails method is one TS method".

The helper also carries the `resolved === undefined` -> `TypeError` throw whose
own convergence is tracked by the closed story
`aggregate-mapping-miss-typeerror-vs-nomethoderror`; check that first, since
inlining moves the throw site.

## Converged shape

Inline the two Rails lines at each arm of the setter and delete `_decompose`,
so the setter's body reads as `writer_method`'s does.

## Acceptance criteria

- [ ] `_decompose` is gone from `aggregations.ts`; the setter arms carry Rails'
      `mapping.each` / `@aggregation_cache[name] = part.dup.freeze` inline.
- [ ] `aggregations.test.ts` and `multiparameter-attributes.test.ts` green.
