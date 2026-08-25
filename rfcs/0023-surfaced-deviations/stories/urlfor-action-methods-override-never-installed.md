---
title: "UrlFor's action_methods override is ported but never installed"
status: ready
updated: 2026-07-27
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "actionpack"
deps: []
deps-rfc: []
est-loc: 140
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Rails narrows a routed controller's action list by overriding
`action_methods` — `vendor/rails/actionpack/lib/abstract_controller/url_for.rb`:

```ruby
module ClassMethods
  def _routes; raise "..."; end
  def action_methods
    @action_methods ||= super - _routes.named_routes.helper_names
  end
end
```

so `posts_path` / `post_url` helper methods can't be dispatched to as actions.

Trails ports the subtraction as a **pure function** —
`actionMethods(baseActionMethods, routes)` in
`packages/actionpack/src/abstract-controller/url-for.ts` — but **nothing
installs it as an override.** `AbstractController.actionMethods()`
(`base.ts`) walks the prototype chain and memoizes, never consulting
`_routes`. Verified: outside `url-for.test.ts` nothing in `packages/` calls it;
the barrel re-exports it and that is the end of the chain.

Consequences on a controller with routes wired:

- `Controller.actionMethods()` still lists named-route helper names.
- `available_action?` / `action_method?` therefore answer `true` for them, so
  a request for `/posts_path` dispatches into the URL helper instead of
  raising `ActionNotFound`.

Found while closing `extra-surface-abstractcontroller-apply-mixin-pattern`
(PR #5332). That PR inlined the helper's body into the Rails-named
`actionMethods` and left a caveat comment on `base.ts`'s `isActionMethod`
noting that it probes the memoized Set, so an override that filters _around_
the memo would not narrow the predicate — that comment is the seam this story
has to work with.

Note `action-controller/metal/flash.ts:45` already has a
`actionMethods(superMethods: Set<string>)` narrowing hook of the same shape,
also not consulted by `AbstractController.actionMethods()`. Whatever mechanism
this story picks should cover both, since Rails composes them by ordinary
`super` chaining through included modules.

## Acceptance criteria

- A controller with `_routes` wired excludes named-route helper names from
  `actionMethods()`, and `isActionMethod` / `isAvailableAction` agree with it
  (i.e. the narrowing lands in the memo, not around it — see the caveat on
  `base.ts` `isActionMethod`).
- The `flash.ts` narrowing hook composes through the same mechanism rather
  than getting a second bespoke one.
- Rails' memoization shape is preserved: the subtraction happens once and is
  invalidated by `clearActionMethodsBang` / `methodAdded`.
- Tests ported from
  `vendor/rails/actionpack/test/controller/url_for_test.rb` and
  `abstract/abstract_controller_test.rb` where they cover this, with names
  matching Rails verbatim; at least one asserts a named-route helper is NOT
  dispatchable as an action.
- `url-for.ts`'s `actionMethods` is either the installed override or deleted
  in favor of one — it must not remain an uncalled pure function.
