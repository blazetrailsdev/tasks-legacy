---
title: "CurrentAttributes subclasses do not inherit class-level method_missing dispatch"
status: draft
updated: 2026-08-18
rfc: "0023-surfaced-deviations"
cluster: null
packages:
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

# Context

Split off `current-attributes-port-body` after PR #6690 landed most of its
bullets (`NOT_SET`, the `defaults` class attribute, `resolve_defaults`,
`attributes=`, `INVALID_ATTRIBUTE_NAMES` + `ArgumentError`, the singleton
accessors, and the removal of `_definitions` / `_get` / `_set` / `_instances` /
`_setupProxy`). This is the class-level-dispatch bullet, which did not converge.

Rails puts three hooks on `CurrentAttributes`' singleton class, so **every**
subclass answers **every** public instance method off the class itself:

- `method_missing(name, ...)` → `instance.public_send(name, ...)`
  (`vendor/rails/activesupport/lib/active_support/current_attributes.rb:178-180`)
- `respond_to_missing?(name, _)` → `instance.respond_to?(name) || super`
  (`current_attributes.rb:182-184`)
- `method_added(name)` → `Delegation.generate(singleton_class, [name], to: :instance, as: self, nilable: false)`
  (`current_attributes.rb:186-193`), the eager form of the same thing.

`packages/activesupport/src/current-attributes.ts` implements none of them.
`attribute()` defines class-level accessors for _declared attributes only_
(mirroring `Delegation.generate` at `current_attributes.rb:138-139`), so
`Current.world` works but `Current.intro`, `Current.time_zone`,
`Current.set_world_and_account` and `Current.respond_to?(:foo)` do not.

The workaround PR #6690 used is per-subclass wrapping at the declaration site,
in `packages/activesupport/src/current-attributes.test.ts`:

```ts
const Current = methodMissingProxy(CurrentClass, {
  delegate: (klass) => klass.instance(),
}) as typeof CurrentClass & InstanceType<typeof CurrentClass>;
```

That is the settled trails idiom for `method_missing`
(`packages/activesupport/src/method-missing-proxy.ts`), and it works, but it is
opt-in per subclass: a `class Foo extends CurrentAttributes {}` declared the
plain way silently loses class-level dispatch, where in Ruby it is inherited.
Every real application subclass would have to remember the wrap.

Note the coupling that made this bite: a `Proxy` and its target are distinct
object identities, so any per-class store keyed by the constructor gives them
two different singletons. PR #6690 fixed that by converging `instance` onto
Rails' `current_instances[current_instances_key]` (`:101-103`, `:170-178`),
keyed by the class _name_. Keep that property — a future change back to an
identity-keyed store re-breaks the proxy.

## Converged shape

Class-level dispatch is inherited, not opt-in: declaring
`class Current extends CurrentAttributes {}` is enough for `Current.<anything
the instance answers>` to resolve, as `current_attributes.rb:178-193` gives it.

If TypeScript genuinely cannot make the wrap inherited (a subclass declaration
produces a bare constructor with no hook to intercept), the fallback is a
single documented entry point in `current-attributes.ts` that subclasses go
through — not a wrap open-coded at each declaration site — carrying the
`current_attributes.rb:178-193` citation.

## Acceptance criteria

- [ ] `Current.<public instance method>` and `"<name>" in Current` resolve for
      a subclass declared without any per-site proxy wrapping, matching
      current_attributes.rb:178-184.
- [ ] `current-attributes.test.ts` declares `Current` as a plain
      `class Current extends CurrentAttributes` and still asserts against the
      class receiver in `using keyword arguments`, `delegation`,
      `all methods forward to the instance` and
      `respond_to? for methods that have not been called`.
- [ ] `current_attributes_test.rb` stays at 0/0/0 in
      `pnpm parity:test -- --assertions --package activesupport`.
- [ ] `pnpm parity:api:extra --package activesupport` gains no novel name.
