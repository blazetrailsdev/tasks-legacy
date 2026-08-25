---
title: "Application#initializers drops railties_initializers, so config.railties_order is ignored"
status: draft
updated: 2026-08-17
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "trailties"
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

## Context

`Trails::Application#initializers` does not call `railties_initializers`, which
Rails' does.

`vendor/rails/railties/lib/rails/application.rb:445-449`:

```ruby
def initializers # :nodoc:
  Bootstrap.initializers_for(self) +
  railties_initializers(super) +
  Finisher.initializers_for(self)
end
```

`railties_initializers` (`application.rb:614`) walks `ordered_railties` so
`config.railties_order` can reorder engines around the app. trails'
`packages/trailties/src/application.ts` (`get initializers()`) splices the
inherited collection in load order instead:

```ts
const bootstrap = Bootstrap.initializersFor(this);
const inherited = super.initializers;
return bootstrap.plus(inherited).plus(Finisher.initializersFor(this));
```

This was tracked by a `@missingRailsCall railties_initializers` JSDoc tag until
PR #6656. That PR made the extractor record call sets for `get` accessors; once
the getter's own call set became visible, the call gate stopped flagging the
call and reported the tag as STALE. A justification is only-shrink exactly like
a baseline row, so the tag had to go — but **the deviation did not converge**,
it merely became invisible to the gate. The prose was kept at the call site
saying so. This story exists to actually close it.

Note the gate cannot currently re-detect this one, so convergence has to be
driven from the Rails source rather than from a red gate.

## Converged shape

Port `ordered_railties` (`application.rb:592-612`) and `railties_initializers`
(`application.rb:614-628`), then spell the body as Rails does —
`Bootstrap.initializersFor(this)`, `railtiesInitializers(super.initializers)`,
`Finisher.initializersFor(this)` — so `config.railties_order` is honoured.
Remove the deviation paragraph from the getter's JSDoc rather than rewording it.

If `ordered_railties` genuinely cannot be ported yet, `pnpm tasks block` this
with the specific blocker; "it would be a bigger diff" is not one.

## Acceptance criteria

- [ ] `Application#initializers` calls `railtiesInitializers(super.initializers)`,
      matching `application.rb:445-449`.
- [ ] `ordered_railties` / `railties_initializers` are ported at their Rails
      names, with `config.railties_order` respected.
- [ ] The deviation paragraph is deleted from the getter's JSDoc, not reworded.
- [ ] `pnpm parity:api` / `pnpm parity:api:calls` deltas non-negative.
