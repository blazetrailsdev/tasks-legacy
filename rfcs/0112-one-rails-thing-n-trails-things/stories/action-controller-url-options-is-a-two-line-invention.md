---
title: "ActionController::UrlFor#url_options is a two-line invention beside a free urlOptionsFromRequest"
status: draft
updated: 2026-08-25
rfc: "0112-one-rails-thing-n-trails-things"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 160
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`ActionController::UrlFor` (`actionpack/lib/action_controller/metal/url_for.rb`)
is a Concern with exactly three members: `extend ActiveSupport::Concern`,
`include AbstractController::UrlFor`, `def initialize(...)` (which nils
`@_url_options`), and `def url_options` (rb:37-64) — a memoized hash merged over
`super`, then conditionally re-derived for the engine / `original_script_name`
cases and frozen.

trails' `packages/actionpack/src/action-controller/metal/url-for.ts` instead
exports:

- `urlOptionsFromRequest(request)` — no Ruby counterpart at all; a free function
  that builds the `{host:, port:, protocol:, _recall:}` literal `url_options`
  builds inline at rb:38-43.
- `UrlForHost` — a structural host interface for that free function.
- `urlOptions(this: UrlForHost)` — a two-line body (`if (this.request) ... else
return { host: "localhost", protocol: "http" }`) that has none of rb:45-63:
  no `@_url_options` memo, no `merge!(super)`, no `freeze`, no `same_origin` /
  `engine_script_name` / `original_script_name` arm, and a `localhost`/`http`
  default Rails does not have.

Surfaced while deleting the duplicate `action-dispatch/url-for.ts` in
[[converge-duplicate-url-options-and-url-for]] (PR #7049), which removed this
file's `urlFor` re-export line and left the rest untouched as out of scope.

## Converged shape

- `urlOptions` mirrors rb:37-64 line for line, including the `@_url_options`
  memo (`_urlOptions`), the merge over the `AbstractController::UrlFor` half,
  and the three-way `same_origin` / `engine_script_name` /
  `original_script_name` branch that produces `:script_name` or
  `:original_script_name`.
- `initialize` mirrors rb:32-35.
- `urlOptionsFromRequest` and `UrlForHost` are inlined into it and deleted, or
  the host interface survives only as the mixin's `this` type (the settled
  trails idiom) with no free function beside it.
- Note `:original_script_name` is what `route_set.rb:869-873` consumes; the
  UrlForTest case covering it is skipped pending that port
  ([[route-set-trailing-slash-propagation]] covers the sibling gap).

## Acceptance criteria

- [ ] `pnpm parity:api:extra --package actionpack` reports no novel name in
      `action-controller/metal/url-for.ts`.
- [ ] `urlOptions` carries the memo, the merge, and all three script-name arms.
