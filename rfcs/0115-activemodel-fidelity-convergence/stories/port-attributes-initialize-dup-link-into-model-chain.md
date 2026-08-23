---
title: "Port Attributes#initialize_dup into Model's dup chain"
status: done
updated: 2026-08-23
rfc: "0115-activemodel-fidelity-convergence"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 160
pr: 6932
claim: "2026-08-23T17:56:07Z"
assignee: "sweep-trails-only-test-files-onto-trails-name"
blocked-by: null
closed-reason: null
---

## Context

trails#6802 built ActiveModel's `initialize_dup` / `init_internals` prepend chain
on `Model.prototype` from two links, Validations
(`vendor/rails/activemodel/lib/active_model/validations.rb:310-313` and
`:467-471`) and Dirty (`dirty.rb:248-251` and `:372-376`) — see the `prepend()`
calls at the bottom of `packages/activemodel/src/model.ts`.

`ActiveModel::Attributes#initialize_dup`
(`vendor/rails/activemodel/lib/active_model/attributes.rb:111-114`) is a third
link and is absent:

```ruby
def initialize_dup(other) # :nodoc:
  @attributes = @attributes.deep_dup
  super
end
```

`ActiveModel::Model` includes `ActiveModel::API` which includes
`ActiveModel::Attributes`, so in Ruby this link runs for every model — plain
ActiveModel and ActiveRecord alike. In trails the deep-dup is instead performed
by each caller: `Model#dup` (`model.ts` ~:2154-2161) and AR's `dup`
(`packages/activerecord/src/persistence.ts` ~:1355). That duplication is what
makes the whole AM dup chain inert on the AR path (see
`route-ar-dup-attribute-duplication-through-the-initialize-dup-links`).

## Converged shape

`attributes.ts` exports an `initializeDup(super_, other)` opening with the
deep-dup and prepended onto `Model.prototype` in include order — below
Validations and Dirty, matching `include ActiveModel::API` preceding them. The
callers' hand-rolled `_attributes.deepDup()` goes away.

Best sequenced WITH or AFTER
`route-ar-dup-attribute-duplication-through-the-initialize-dup-links`, since the
AR half of the duplicate lives in that story's blast radius; landing this one
alone leaves AR still deep-duping in `dup`.

## Acceptance criteria

- [ ] `Attributes#initialize_dup` is a link in `Model.prototype`'s chain, opening
      with `super_`.
- [ ] No caller performs `_attributes.deepDup()` as part of `dup` any more.
- [ ] Removing the link reds a test.
- [ ] activemodel + activerecord dup-related suites stay green on all four
      adapter lanes.
