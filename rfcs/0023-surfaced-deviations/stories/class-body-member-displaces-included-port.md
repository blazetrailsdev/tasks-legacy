---
title: "Guard against a class-body member silently displacing an include()-mixed port"
status: draft
updated: 2026-08-21
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
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

# Guard against a class-body member silently displacing an `include()`-mixed port

## Context

`include()` deliberately never replaces a class-body method — Ruby's `include`
does not either, and the check is explicit at
`packages/activesupport/src/include.ts:273`:

```ts
if (Object.prototype.hasOwnProperty.call(klass.prototype, key) && !installed.has(key)) {
  continue;
}
```

The consequence is silent and expensive. `packages/activerecord/src/base.ts`
mixes ~60 instance methods onto `Base.prototype` from
`attribute-methods.ts` / `attribute-methods/dirty.ts`. Adding a class-body method
with any of those names wins, the port is never installed, and nothing — not
`tsc`, not `parity:api`, not `parity:api:calls` — says so.

Measured on #6821: a class-body `attributeInDatabase` on `Base` displaced the
port in `attribute-methods/dirty.ts`. Rails' `attribute_in_database` reads
`mutations_from_database.original_value`, i.e. the pending-change baseline
(`vendor/rails/activerecord/lib/active_record/attribute_methods/dirty.rb:164-166`),
while the displaced-in body read `attribute_was`. The only symptom was one
unrelated-looking red — `has-many-associations.test.ts` "counter cache updates in
memory after update with inverse of enabled" — and it cost a bisect to attribute.

The same hazard is why `base.ts` carries prose comments in its include block
("… overriding breaks tests. Category A") instead of a mechanical check.

## Converged shape

A check that fails when a class-body member name collides with a name the
class's own `include()` calls would install. The include machinery already knows
both sets at wire time, so the cheapest form is a dev-time assertion in
`include()` itself: when a key is skipped because the class body owns it, that is
either intentional (Ruby semantics) or a displaced port, and the two are
distinguishable by whether the class-body member is annotated.

An ESLint rule over the `base.ts` include object is the alternative; prefer the
runtime check, which needs no manifest and cannot go stale.

## Acceptance criteria

- [ ] A class-body member that shadows an `include()`d member fails loudly
      (test or lint), naming both the class-body site and the displaced port.
- [ ] Intentional Ruby-semantics shadowing has an explicit opt-out annotation,
      and every existing intentional case in `base.ts` carries it — the prose
      "Category A" comments in the include block are replaced by that marker.
- [ ] A regression case pins the measured instance: a class-body
      `attributeInDatabase` on `Base` is reported rather than silently winning.
