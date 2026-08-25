---
title: "RouteSet does not propagate :trailing_slash or :original_script_name to Http::URL"
status: ready
updated: 2026-08-25
rfc: "0112-one-rails-thing-n-trails-things"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 180
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`:trailing_slash` on a non-blank path is applied by the route set, not by
`ActionDispatch::Http::URL`:

- `route_set.rb:218` — `path << "/" if options[:trailing_slash] && !path.end_with?("/")`
- `route_set.rb:882` — `if options[:trailing_slash] && !options[:format] && !path.end_with?("/")`
- `url.rb:76` — Http::URL's own arm is only `path = "/" if options[:trailing_slash] && path.blank?`

trails ports the `url.rb:76` arm (`action-dispatch/http/url.ts`, `pathFor`) but
nothing ports the route-set arm, so `trailingSlash: true` on `/posts` yields
`/posts`.

Surfaced while converging the duplicate `urlFor`
([[converge-duplicate-url-options-and-url-for]], RFC 0112): the deleted
`action-dispatch/url-for.ts` had its own non-Rails-layout implementation of the
route-set behaviour, and the five `trailing slash*` cases in
`action-controller/controller/url-for.test.ts` were the only coverage of it.
They are now pending stubs citing `route_set.rb:882`.
`action-dispatch/dispatch/url-generation.test.ts:239-256` already carries
commented-out assertions for the same gap.

## Acceptance criteria

- [ ] `RouteSet`'s url/path generation appends the trailing slash per
      `route_set.rb:218` and `:882` (including the `!options[:format]` guard).
- [ ] The five `trailing slash*` cases in
      `action-controller/controller/url-for.test.ts` are real assertions again
      (they are `it.skip` as of PR #7049), along with `using nil script name
    properly concats with original script name`, which covers the sibling
      `original_script_name` arm (`route_set.rb:869-873`).
- [ ] The commented-out assertions at
      `action-dispatch/dispatch/url-generation.test.ts:239-256` are restored.
