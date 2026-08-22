---
title: "NullSession#handle_unverified_request writes controller state instead of the request"
status: ready
updated: 2026-08-22
rfc: "0112-one-rails-thing-n-trails-things"
cluster: duplicate-bodies
packages: []
deps: []
deps-rfc: []
est-loc: 250
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`ActionController::RequestForgeryProtection::ProtectionMethods::NullSession#handle_unverified_request`
is (`vendor/rails/actionpack/lib/action_controller/metal/request_forgery_protection.rb:261-267`):

    def handle_unverified_request
      request = @controller.request
      request.session = NullSessionHash.new(request)
      request.flash = nil
      request.session_options = { skip: true }
      request.cookie_jar = NullCookieJar.build(request, {})
    end

Every write goes through the REQUEST. trails'
`packages/actionpack/src/action-controller/metal/request-forgery-protection.ts`
writes `controller.session` / `controller.cookies` instead, drops
`session_options = { skip: true }` entirely, and clears the flash by duck-typed
`flash.clear()` rather than assigning `nil`. #6697 converged the one call-argument
row here (`NullSessionHash.new(request)`) and deliberately left the rest.

The two null classes are also stubs rather than ports:

- `NullSessionHash` (`request_forgery_protection.rb:270-287`) subclasses
  `Rack::Session::Abstract::SessionHash`, calls `super(nil, req)` and sets
  `@data = {}` / `@loaded = true`; trails' class stores nothing (its ctor
  parameter is `_req`) and re-implements `get`/`set`/`has`/`delete`/`clear`
  by hand. Rails' overrides are exactly three: `destroy` (no-op), `exists?`
  (true), `enabled?` (false).
- `NullCookieJar` (`request_forgery_protection.rb:289-293`) subclasses
  `ActionDispatch::Cookies::CookieJar` and overrides only `write(*)`; trails'
  is a standalone class with hand-written `get`/`set`/`has`/`delete` and no
  `build`. The missing `build` is the standing
  `handle_unverified_request` → `build` row in
  `call-mismatches-exclude/actioncontroller/metal/request-forgery-protection.json`.

## Acceptance criteria

- [ ] `handleUnverifiedRequest` writes through the request as Rails does, in
      Rails' statement order, including `session_options = { skip: true }` and
      `flash = nil`.
- [ ] `NullSessionHash` and `NullCookieJar` derive from the session-hash /
      cookie-jar classes they extend in Rails and override only the members
      Rails overrides; `NullCookieJar.build(request, {})` exists and is called.
- [ ] The `handle_unverified_request` → `build` row is deleted from
      `call-mismatches-exclude/actioncontroller/metal/request-forgery-protection.json`.
- [ ] `pnpm parity:api:calls` and `pnpm parity:api:calls:args` stay green.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
