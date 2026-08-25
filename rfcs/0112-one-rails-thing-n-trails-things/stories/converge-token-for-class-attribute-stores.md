---
title: "converge-token-for-class-attribute-stores"
status: claimed
updated: 2026-08-25
rfc: "0112-one-rails-thing-n-trails-things"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
pr: null
claim: "2026-08-25T17:17:28Z"
assignee: "converge-token-for-class-attribute-stores"
blocked-by: null
closed-reason: null
---

# Converge token_for's two class_attribute stores onto classAttribute()

## Context

Split out of `converge-remaining-activerecord-copy-on-write-stores-onto-class-attribute`
(RFC 0112), which converged `_attr_readonly`, `default_scopes` /
`default_scope_override`, `skip_time_zone_conversion_for_attributes` /
`time_zone_aware_types`, `nested_attributes_options` and `encrypted_attributes`
but ran out of LOC budget before this cluster.

`token_for.rb:10-11` declares two class_attributes:

```ruby
class_attribute :token_definitions, instance_accessor: false, instance_predicate: false, default: {}
class_attribute :generated_token_verifier, instance_accessor: false, instance_predicate: false
```

`packages/activerecord/src/token-for.ts` implements both with hand-rolled
WeakMap registries plus prototype-chain walks that re-derive `class_attribute`
read semantics:

- `tokenDefinitionRegistry` (WeakMap) + `resolvedDefinitions()` (token-for.ts:159-181)
  behind `tokenDefinitions()` / `setTokenDefinitions()` (token-for.ts:229-247).
- `tokenVerifierRegistry` (WeakMap) + `ownVerifierEntry()` / `resolvedVerifier()`
  (token-for.ts:17, 47-70) behind `generatedTokenVerifier()` /
  `setGeneratedTokenVerifier()` (token-for.ts:257-278).
- `base.ts` exposes both as getter/setter pairs delegating to those functions.

`classAttribute()` from `@blazetrails/activesupport` already gives exactly these
semantics (reads walk the constructor chain, writes are local), so both
registries and both chain walks should go, the way `_attrReadonly` and
`nestedAttributesOptions` did.

### The one real wrinkle

`_defaultVerifier` (token-for.ts:22) is the boot-time value Rails' railtie
initializer assigns onto `ActiveRecord::Base`
(`railtie.rb:328-334`, `self.generated_token_verifier ||= app.message_verifier(...)`).
trails has no railtie, so `setTokenForSecret()` (a declared trails-only seam)
rebuilds it into a module-level slot that the reader falls back to _after_ the
chain walk. Once `generatedTokenVerifier` is a `classAttribute` on `Base` there
is no post-chain fallback: the boot value has to be _assigned onto `Base`_, which
`token-for.ts` cannot do directly — `base.ts` imports `token-for.ts` at runtime,
so a reverse runtime import closes a cycle.

Options to weigh (pick one, cite it at the call site):

1. Assign in `base.ts` right after the `classAttribute.call(Base, ...)` line, and
   give `setTokenForSecret` a registered callback that `base.ts` installs — an
   extension of the already-`@noRailsEquivalent` railtie seam.
2. A zero-import slot module (CLAUDE.md, _Call-time constant resolution_) if the
   cycle is genuinely unavoidable.

Also note the trails-only `TokenDefinitionsHash` `fetch`/`merge` wrapper
(token-for.ts:184-213) — it must survive as the `classAttribute` default and be
re-applied by the writer, or every `token_definitions.fetch(purpose)` call site
breaks.

## Acceptance criteria

- [ ] `tokenDefinitions` and `generatedTokenVerifier` are `classAttribute()`
      declarations on `Base` with Rails' `instance_accessor: false,
instance_predicate: false`.
- [ ] `tokenDefinitionRegistry`, `tokenVerifierRegistry`, `resolvedDefinitions`,
      `resolvedVerifier` and `ownVerifierEntry` are deleted; no
      copy-on-first-write helper survives under any name.
- [ ] The boot-time verifier seam is documented at its call site with the
      `railtie.rb:328-334` citation.
- [ ] `packages/activerecord/src/token-for*.test.ts` green; `pnpm parity:api` /
      `pnpm parity:test` deltas non-negative.
