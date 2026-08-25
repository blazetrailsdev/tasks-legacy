---
title: "Register controllers in the constant table so controllerClassFor can constantize instead of throwing"
status: draft
updated: 2026-07-28
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "actionpack"
  - "activesupport"
deps: []
deps-rfc: []
est-loc: 220
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`ActionDispatch::Request#controller_class_for` resolves a controller by name:

```ruby
def controller_class_for(name)
  if name
    controller_param = name.underscore
    const_name = controller_param.camelize << "Controller"
    begin
      const_name.constantize
    rescue NameError => error
      ...
```

(`vendor/rails/actionpack/lib/action_dispatch/http/request.rb:94-110`)

trails throws unconditionally instead
(`packages/actionpack/src/action-dispatch/http/request.ts:961`), and its doc
comment gives the reason:

> Trails has no global constant table to back Rails'
> `"#{name.camelize}Controller".constantize` lookup, so callers must resolve
> the class through the router (which knows the registered controllers for a
> route) until that bridge lands.

**That justification is now out of date.** PR #5471 ported
`ActiveSupport::Inflector.constantize` with a real constant table
(`packages/activesupport/src/inflector.ts`). What is missing is not the
table — it is that controllers never register into it, unlike AR models,
which now do via `registerConstant` at `Base.adapter=` / `registerModel` /
`registerSubclass`.

`actioncontroller/test-case.ts:84` has the same shape, resolving
`<Name>Controller` off `globalThis` as an ad-hoc stand-in.

Both sites are baselined in the wide call-mismatch ratchet by #5471
(`scripts/api-compare/call-mismatches-wide-exclude/actiondispatch/http/request.json`
and `.../actioncontroller/test-case.json`) with reasons that this story
should make obsolete.

## Acceptance criteria

- Controllers register into the Inflector constant table at definition, the
  way AR models do — pick the analogue of AR's registration seam
  (`ActionController::Base` inherited hook / an explicit registration call)
  and tag it `@noRailsEquivalent` like `registerConstant` is.
- `controllerClassFor` resolves through `constantize` and reproduces Rails'
  `NameError` rescue arms (`request.rb:99-110`), rather than throwing
  unconditionally. Its stale doc comment goes away.
- `test-case.ts`'s `tests(name)` stops reaching for `globalThis`.
- The two corresponding entries are REMOVED from
  `call-mismatches-wide-exclude/`, not reworded.
- Verify no import cycle: actionpack already depends on activesupport, but
  confirm the registration seam does not pull the controller base into a
  module-init cycle (see the AR precedent at `base.ts` — the globalid wire is
  a side-effect import at the foot of the file for exactly this reason).

## Triage note (2026-08-18): the baseline path in this body is stale

This story cites `scripts/api-compare/call-mismatches-wide-exclude/…`. **That
tree no longer exists.** RFC 0084 folded the narrow RFC 0044 ratchet and the
wide one into a single gate over a single baseline:
`scripts/api-compare/call-mismatches-exclude/<package>/<tsFile .ts→.json>`,
gated by `pnpm parity:api:calls` (call-set rows) and `pnpm parity:api:calls:args`
(`kind: "args"` rows, RFC 0095).

Look for the row there, under the same `rubyName` / `call` pair. Everything else
in this story — the Rails and trails `file:line` citations, the described
divergence — is unaffected; only the path to the baseline row changed.

Remember the baseline is only-shrink: on converging, delete the one row by hand
(via `serializeBaseline`, sorted — never `--write`/reseed), then
`pnpm parity:api:calls:tighten <package>/<file>.json` for the stale high-water mark.
