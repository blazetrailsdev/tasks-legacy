---
title: "camelCase the rack cookie-header option keys and drop the actionpack bridge"
status: draft
updated: 2026-08-17
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "actionpack"
  - "rack"
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

`Rack::Utils.set_cookie_header` / `delete_set_cookie_header!` are ported in
packages/rack/src/utils.ts:286-364, but their cookie option hash is read with
Ruby snake_case keys — `opts.max_age`, `opts.same_site`, `opts.http_only`,
`opts.httponly` — where docs/ruby-ts-conventions.md translates Ruby symbol
option keys to camelCase (`maxAge`, `sameSite`, `httpOnly`).

Rails source: rack-3.1.12 `rack/lib/rack/utils.rb:294-400`
(`set_cookie_header`, `delete_set_cookie_header`, `delete_set_cookie_header!`),
whose Ruby keys are `:domain, :path, :max_age, :expires, :secure, :httponly,
:http_only, :same_site, :partitioned, :value`.

PR #6671 hit this at the actionpack boundary: `Response#setCookie` /
`#deleteCookie`
(packages/actionpack/src/action-dispatch/http/response.ts) hold camelCased
`CookieOptions`, and Rails passes its option hash straight to
`Rack::Utils.set_cookie_header`. Because the key spellings disagree, the port
needs a private `rackCookieValue()` bridge that exists only to rename keys —
surface Rails does not have.

## Converged shape

Rename the option keys in packages/rack/src/utils.ts to the conventional
camelCase spellings (keeping every Ruby arm, including the
`"httponly" in opts ? opts.httponly : opts.http_only` fallback pair), update
rack's own callers/tests, then delete `rackCookieValue` from response.ts so
`setCookie` passes its `CookieOptions` through unchanged, as Rails does.

## Acceptance criteria

- [ ] rack's cookie-header utils read camelCase option keys per docs/ruby-ts-conventions.md.
- [ ] `rackCookieValue` is deleted; `Response#setCookie`/`#deleteCookie` pass the options through.
- [ ] `pnpm parity:api:calls` / `:args` stay green.
