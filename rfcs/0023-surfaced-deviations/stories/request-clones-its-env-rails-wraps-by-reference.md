---
title: "ActionDispatch::Request clones its env; Rails wraps it by reference"
status: draft
updated: 2026-08-18
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "actionpack"
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

# `ActionDispatch::Request` clones its env; Rails wraps it by reference

## Context

Rails' `ActionDispatch::Request.new(env)` keeps `@env` as the SAME object the
Rack middleware stack passes down, which is what makes `set_header` visible to
every later middleware. trails' `Request` clones its env instead — recorded at
`packages/actionpack/src/action-dispatch/middleware/host-authorization.ts:278`
("the `Request` constructed in `call` clones its env").

The cost is concrete. `host_authorization.rb:167` is

```ruby
def mark_as_authorized(request)
  request.set_header("action_dispatch.authorized_host", request.host)
end
```

The port cannot converge to that one-argument signature — a `request.setHeader`
write lands on the clone and never reaches the caller's env — so it threads the
raw `env` as an extra first argument and writes
`env["action_dispatch.authorized_host"] = stripPort(request.rawHostWithPort)`,
re-deriving by hand what `request.host` (`http/url.rb`) already answers. That is
the `call -> mark_as_authorized` `kind: "args"` row in
`scripts/api-compare/call-mismatches-exclude/actiondispatch/middleware/host-authorization.json`.
Attempted in PR #6687 and reverted: the convergence reds
`dispatch/host-authorization.test.ts:235,424` until the clone goes.

Any other ported body that writes through `request.set_header` and expects a
downstream middleware to read it has the same latent bug.

## Acceptance criteria

- [ ] `Request` holds the env by reference, as Rails does.
- [ ] `markAsAuthorized` takes only `request` and writes
      `request.setHeader("action_dispatch.authorized_host", request.host)`; its
      baseline row is DELETED by hand (only-shrink, no reseed), and the local
      `stripPort` helper goes with it.
- [ ] Audit the other `setHeader` writers for reads that were compensating for
      the clone.
