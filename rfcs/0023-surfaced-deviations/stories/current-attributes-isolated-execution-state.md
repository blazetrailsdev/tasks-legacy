---
title: "CurrentAttributes singletons are module-level, not IsolatedExecutionState-scoped; reset_all/clear_all unported"
status: draft
updated: 2026-08-18
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activesupport"
deps: []
deps-rfc: []
est-loc: 150
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

# Context

Split off `current-attributes-port-body` after PR #6690 landed most of its
bullets. This is the execution-isolation bullet, which did not converge.

Rails scopes the per-class singleton to the current execution context:

```ruby
def instance
  current_instances[current_instances_key] ||= new
end                                    # current_attributes.rb:101-103

def current_instances
  IsolatedExecutionState[:current_attributes_instances] ||= {}
end                                    # current_attributes.rb:170-172

def current_instances_key
  @current_instances_key ||= name.to_sym
end                                    # current_attributes.rb:174-176
```

`IsolatedExecutionState` is what makes `Current` a _per-request_ singleton and
what its `isolation_level` (`:fiber` / `:thread`) switches.

PR #6690 converged the key and the lookup shape — `instance` now resolves
`currentInstances.get(currentInstancesKey())` keyed by the class name, per
`:101-103` and `:174-176` — but `currentInstances` in
`packages/activesupport/src/current-attributes.ts` is a module-level `Map`, not
`IsolatedExecutionState`, so there is no execution isolation at all. The file
header says so explicitly rather than claiming otherwise.

Two consequences, both visible in
`packages/activesupport/src/current-attributes.test.ts`:

- `CurrentAttributes use fiber-local variables` and
  `CurrentAttributes can use thread-local variables` (current_attributes_test.rb:216-240)
  are `it.skip` stubs. They are the tests that flip
  `ActiveSupport::IsolatedExecutionState.isolation_level` and assert the
  singleton is / is not shared across an `Enumerator`.
- `reset_all` and `clear_all` (`current_attributes.rb:155-160`) are unported.
  Both iterate `current_instances`, so they belong with this change:

  ```ruby
  def reset_all
    current_instances.each_value(&:reset)
  end

  def clear_all
    reset_all
    current_instances.clear
  end
  ```

Note the constraint PR #6690 established: a `Proxy` and its target are distinct
object identities, so the store must stay keyed by the class _name_ rather than
the constructor — an identity-keyed store re-breaks the `methodMissingProxy`
wrap tracked by `current-attributes-class-level-method-missing`.

## Converged shape

`currentInstances` reads through `IsolatedExecutionState` under the
`:current_attributes_instances` key (`current_attributes.rb:170-172`) rather
than a module-level `Map`, and `resetAll` / `clearAll`
(`current_attributes.rb:155-160`) are ported over it.

## Acceptance criteria

- [ ] `currentInstances` resolves through `IsolatedExecutionState`
      (current_attributes.rb:170-172); the store stays keyed by
      `currentInstancesKey()`, i.e. the class name, not the constructor.
- [ ] `resetAll` and `clearAll` ported per current_attributes.rb:155-160.
- [ ] `CurrentAttributes use fiber-local variables` and
      `CurrentAttributes can use thread-local variables` are enabled and pass,
      asserting what current_attributes_test.rb:216-240 asserts.
- [ ] `current_attributes_test.rb` stays at 0/0/0 in
      `pnpm parity:test -- --assertions --package activesupport`, and the
      activesupport `parity:test` percent does not drop.
- [ ] The file header stops describing the store as process-wide.
