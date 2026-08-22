---
title: "_exec_scope takes an explicit fn parameter instead of Rails' instance_exec block"
status: ready
updated: 2026-08-22
rfc: "0112-one-rails-thing-n-trails-things"
cluster: duplicate-bodies
deps: []
deps-rfc: []
est-loc: 160
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced by PR #6121: adding forwarding-parameter support to the prism codegen
made `def _exec_scope(...)` a clean generated def for the first time, which
reded the convergence guard with a pre-existing port divergence. It was
catalogued as one row in `scripts/prism-codegen/convergence-baseline.json`
(`active_record/relation.rb::_execScope::divergent`) to unblock that PR. This
story converges it.

Rails (`vendor/rails/activerecord/lib/active_record/relation.rb:552-558`):

```ruby
def _exec_scope(...) # :nodoc:
  @delegate_to_model = true
  registry = model.scope_registry
  _scoping(nil, registry) { instance_exec(...) || self }
ensure
  @delegate_to_model = false
end
```

trails (`packages/activerecord/src/relation.ts`, `_execScope`):

```ts
_execScope(fn: (rel: Relation<T>, ...args: unknown[]) => unknown, ...args: unknown[]): unknown {
  this._delegateToModel = true;
  const registry = this.model.scopeRegistry();
  try {
    return this._scoping(null, registry, () => fn(this, ...args) || this);
  } finally {
    this._delegateToModel = false;
  }
}
```

Two divergences, both in the block:

1. Rails takes **no explicit callable**: `(...)` forwards the caller's block,
   and `instance_exec` runs it with `self` bound to the relation. trails takes
   `fn` as a leading positional parameter — extra surface Rails does not have.
2. Rails' scope body therefore sees the relation as `self`; trails passes it as
   the block's **first argument** (`fn(this, ...args)`), which is why every
   trails scope body is written `(q) => q.where(...)` rather than Rails'
   `-> { where(...) }`.

## Converged shape

`instance_exec` has no direct TS image, so this needs the settled decision for
Ruby-block-with-rebound-`self` before the body can move. The candidate is a
`this`-bound call (`fn.call(this, ...args)`) plus scope bodies written against
`this`, which would let `_execScope` drop the leading parameter and match the
Rails signature under forwarding params (`(...args)`).

Note the blast radius: changing how a scope body receives the relation touches
every `this.scope(...)` call site in `test-helpers/models/**` and any user-facing
scope docs. Size accordingly — the estimate covers `relation.ts` plus the
scope-body sweep, not a docs pass.

## Acceptance criteria

- `_execScope` takes no explicit callable parameter; the relation reaches the
  block as `self`/`this`, as `instance_exec` gives it (`relation.rb:555`).
- Scope bodies in `packages/activerecord/src/test-helpers/models/**` are updated
  to the decided shape.
- The row `active_record/relation.rb::_execScope::divergent` is **removed** from
  `scripts/prism-codegen/convergence-baseline.json` (only-shrink), not rewritten
  with a better reason.
- AR suite green on all three lanes; `parity:api:extra` shows no new novel surface.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
