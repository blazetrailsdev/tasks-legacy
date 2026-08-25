---
title: "ActiveModel::Name derives its memos by joining pre-split segments instead of inflecting @name"
status: draft
updated: 2026-08-11
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activemodel"
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

`ActiveModel::Name#initialize` derives every one of its memoized names from
`@name` with Inflector calls. trails' constructor instead takes a pre-segmented
namespace array and reassembles the names by joining segments, which changes the
derivation for several fields.

Rails (`activemodel/lib/active_model/naming.rb:173-183`):

```ruby
@singular     = _singularize(@name)
@plural       = ActiveSupport::Inflector.pluralize(@singular, locale)
@uncountable  = @plural == @singular
@element      = ActiveSupport::Inflector.underscore(ActiveSupport::Inflector.demodulize(@name))
@human        = ActiveSupport::Inflector.humanize(@element)
@param_key    = (namespace ? _singularize(@unnamespaced) : @singular)
@route_key          = (namespace ? ActiveSupport::Inflector.pluralize(@param_key, locale) : @plural.dup)
@singular_route_key = ActiveSupport::Inflector.singularize(@route_key, locale)
```

trails today (`packages/activemodel/src/naming.ts:300-339`) builds
`bareUnderscored` / `segmentsUnderscored` and joins them, so e.g. `@singular`
becomes `[...segmentsUnderscored, bareUnderscored].join("_")` rather than
`_singularize(@name)`, and `@i18n_key` becomes a `/`-join rather than
`@name.underscore.to_sym`.

Surfaced by the RFC 0096 activemodel naming burndown (PR #6350) as two `naming`
rows (`pluralize` receives `bareUnderscored` where Rails passes `param_key`;
`underscore` receives `name` where Rails passes `demodulize(@name)`). Deliberately
NOT renamed there — the identifiers are downstream of a restructured derivation,
and renaming them would have asserted a correspondence that does not hold.

The pre-segmentation is likely load-bearing for the TS side (no Ruby
`Module.nesting` / autoload namespace to demodulize against), so this needs a
real look rather than a mechanical revert. That is exactly why it is a story and
not a rename.

## Converged shape

Derive each memo from `@name` through the Inflector calls Rails uses, in Rails'
order, keeping segmentation only where TS genuinely cannot recover the namespace
from the class — and, where it cannot, confining the deviation to the ONE
derivation that needs it rather than to all of them.

## Acceptance criteria

1. `singular`, `plural`, `element`, `human`, `paramKey`, `routeKey`,
   `singularRouteKey` and `i18nKey` are each derived by the Rails expression at
   `naming.rb:173-183`, or the specific derivation that cannot be is documented
   at the call site with what TS is missing.
2. The two `naming` rows for `activemodel/naming.ts` in
   `pnpm parity:api:calls:args:report` are gone or reduced; report before/after.
3. No change to any currently-passing `ModelName` test, and namespaced-model
   cases stay covered.
4. `pnpm parity:test` delta non-negative.
