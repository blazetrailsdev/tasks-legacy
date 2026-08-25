---
title: "Rack::Logger should pass started_request_message as a block, not an eager string"
status: ready
updated: 2026-08-21
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "trailties"
deps: []
deps-rfc: []
est-loc: 90
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`packages/trailties/src/rack/logger.ts#callApp` logs with

```ts
this.logger.info?.(this.startedRequestMessage(request));
```

Rails (`vendor/rails/railties/lib/rails/rack/logger.rb:39`) writes

```ruby
logger.info { started_request_message(request) }
```

The block form is deliberate: `Logger#info` with a block does not evaluate the
message unless the severity is enabled, so `started_request_message` — which
calls `filtered_path` (running the whole parameter filter) and `Time.now` — is
skipped entirely when info logging is off. trails' `RackLoggerLike#info` takes
a `string`, so the port evaluates eagerly on every request.

This is what keeps `rack/logger.ts` `call_app` -> `info` standing as a `naming`
call-argument row (`ref:request` vs the port's computed string), and it is a
real cost, not only a recording artifact.

Surfaced while threading the request through `Rack::Logger`
(PR #6538, RFC 0096).

## Converged shape

`RackLoggerLike#info` accepts `string | (() => string)`, matching Ruby's
`Logger#info(message = nil, &block)`, and `callApp` passes the thunk. The
ActiveSupport logger ports that satisfy the interface resolve a function
argument only after the level check, as `Logger#add` does.

## Acceptance criteria

- [ ] `callApp` passes `() => this.startedRequestMessage(request)`, per
      logger.rb:39.
- [ ] `RackLoggerLike#info` (and the logger implementations behind it) accept
      and lazily resolve a block argument.
- [ ] A test asserts `startedRequestMessage` is not evaluated when the logger
      declines the message.
- [ ] `pnpm parity:api:calls:args:report` shows the `rack/logger.ts`
      `call_app` -> `info` `naming` row retired, with no new `shape` row.

## Parked — out of the AR closure (2026-08-18)

Deliberately left `draft` by the RFC 0096 re-scope. `packages/trailties/src/rack/logger.ts`
is not in the AR require-closure (`scripts/api-compare/ar-closure.ts`), so this
row is one of the 173 out-of-scope naming rows the re-scope hands off rather
than burns down — see `naming-gate-flip` `## Re-scope (2026-08-18)`.

It is not abandoned: `pnpm parity:api:calls:args:report` still measures it, and
it does not gate. Re-home it into an actionpack-family naming RFC when one
exists; do not flip it `ready` under RFC 0096, which now scopes to the closure.

## Re-homed from RFC 0096 (2026-08-21)

RFC 0096 re-scoped to the AR require-closure on 2026-08-18;
`packages/trailties/src/rack/logger.ts` is outside it, so this row is one of the
173 out-of-closure naming rows 0096 hands off rather than burns down. Owner
decision at the 2026-08-21 backlog triage: re-home it here rather than leave it
parked as a permanent `draft` under an RFC that will not schedule it. Move it
into an actionpack-family naming RFC if one is ever created.
