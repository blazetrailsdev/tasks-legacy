---
title: "Converge HostAuthorization#blocked_hosts onto the Rails reader calls"
status: draft
updated: 2026-08-18
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "actionpack"
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

# Converge `HostAuthorization#blocked_hosts` onto the Rails reader calls

## Context

Surfaced while converging `mark_as_authorized`/`excluded?` in PR #6701
(0108-call-gate-false-positives/request-env-by-reference). Out of scope there and
left untouched; it carries no baseline row, so nothing currently tracks it.

Rails, `actionpack/lib/action_dispatch/middleware/host_authorization.rb:150-160`:

    def blocked_hosts(request)
      hosts = []

      origin_host = request.get_header("HTTP_HOST")
      hosts << origin_host unless @permissions.allows?(origin_host)

      forwarded_host = request.x_forwarded_host&.split(/,\s?/)&.last
      hosts << forwarded_host unless forwarded_host.blank? || @permissions.allows?(forwarded_host)

      hosts
    end

trails, `packages/actionpack/src/action-dispatch/middleware/host-authorization.ts:263-276`,
reaches into `request.env` directly instead of through the two reader methods, and
invents a fallback chain Rails does not have:

    const env = request.env;
    const originHost =
      (env["HTTP_HOST"] as string | undefined) ??
      (env["SERVER_NAME"] as string | undefined) ??
      "localhost";
    ...
    const forwardedHeader = env["HTTP_X_FORWARDED_HOST"] as string | undefined;
    const forwarded = forwardedHeader?.split(/,\s?/).pop()?.trim();

Three separate divergences:

1. `env["HTTP_HOST"]` instead of `request.getHeader("HTTP_HOST")`.
2. The `?? SERVER_NAME ?? "localhost"` fallback is invented — Rails passes a `nil`
   `origin_host` straight to `allows?`, so a missing `HTTP_HOST` is a _blocked_
   host in Rails and a silently allowed `localhost` in trails whenever
   `SERVER_NAME` is absent. This is a behavioral gap in a security middleware,
   not just a shape one.
3. `env["HTTP_X_FORWARDED_HOST"]` instead of `request.xForwardedHost`
   (`action_dispatch/http/url.rb`), and a `.trim()` Rails does not do — Rails uses
   `forwarded_host.blank?` for the empty arm instead.

## Acceptance criteria

- [ ] `blockedHosts` calls `request.getHeader("HTTP_HOST")` and
      `request.xForwardedHost`, with no invented `SERVER_NAME` / `"localhost"`
      fallback.
- [ ] The empty-forwarded-host arm uses `blank?` semantics (ActiveSupport
      `isBlank`), matching host_authorization.rb:157.
- [ ] A test covers the env with no `HTTP_HOST` and no `SERVER_NAME` being
      BLOCKED, not allowed (the behavioral half of this story — it must fail on
      the current baseline).
- [ ] `pnpm parity:api:calls` / `:args` green with no new baseline row.
