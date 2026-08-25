---
title: "Drop Request's invented Rack-minimum env defaults in favour of Rack::MockRequest.env_for"
status: draft
updated: 2026-08-18
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "actionpack"
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

# Drop `Request`'s invented Rack-minimum env defaults in favour of `Rack::MockRequest.env_for`

## Context

Surfaced in PR #6701 (0108-call-gate-false-positives/request-env-by-reference),
which made `ActionDispatch::Request` hold its Rack env by reference
(`rack/request.rb:62-65`, `@env = env`). Deliberately left out of scope there and
called out in the PR body.

`Request#initialize` in Rails (`actionpack/lib/action_dispatch/http/request.rb:64-73`)
fills in NO env defaults — it calls `super` and nils out its memos. trails'
constructor (`packages/actionpack/src/action-dispatch/http/request.ts:166-175`)
instead invents seven:

    this.env["REQUEST_METHOD"] ??= "GET";
    this.env["SERVER_NAME"] ??= "localhost";
    this.env["SERVER_PORT"] ??= "80";
    this.env["PATH_INFO"] ??= "/";
    this.env["QUERY_STRING"] ??= "";
    this.env["rack.url_scheme"] ??= "http";
    this.env["rack.input"] ??= "";

These stand in for `Rack::MockRequest.env_for`, which is what Rails' own request
tests build their env through (`actionpack/test/dispatch/request_test.rb:20-39`,
`env = Rack::MockRequest.env_for(uri, env)`) and which trails has not ported.

Two reasons this is now worse than it was: since #6701 the env is held by
reference, so these writes MUTATE the caller's env object — a `new Request(env)`
anywhere in the middleware stack silently injects seven keys into the env the
downstream app receives. And production envs come from a real server that already
supplies them, so the defaults only ever fire for hand-built test envs — exactly
what `env_for` is for.

Removing the block outright reds one test, `host without specifying port`
(`request_test.rb:448` -> host-authorization/request.test.ts), because
`raw_host_with_port` (`url.rb:216-226`) falls back to
`"#{SERVER_NAME}:#{SERVER_PORT}"` and a missing `SERVER_PORT` yields
`"example.com:undefined"`. In Rails that test's env goes through `env_for`, which
supplies `SERVER_PORT`. So the port of `env_for` is the prerequisite, not an
optional extra.

## Acceptance criteria

- [ ] `Rack::MockRequest.env_for` ported (rack/lib/rack/mock_request.rb) far
      enough for the actionpack request tests to build their envs through it.
- [ ] Trails' request tests that hand-build an env route through it, mirroring
      `stub_request` (`request_test.rb:20-39`).
- [ ] The seven `??=` default writes are DELETED from `Request`'s constructor, so
      the constructor body matches request.rb:64-73 and no longer mutates the
      caller's env with keys it did not set.
- [ ] `host without specifying port` (`request_test.rb:448`) still passes, now
      because `env_for` supplied `SERVER_PORT`.
- [ ] `pnpm parity:api:extra --package actionpack` shows no new surface;
      `pnpm vitest run packages/actionpack` green.
