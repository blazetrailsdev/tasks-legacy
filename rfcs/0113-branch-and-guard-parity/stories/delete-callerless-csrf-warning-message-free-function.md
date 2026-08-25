---
title: "Delete the callerless warningMessage(origin, baseUrl) free function"
status: ready
updated: 2026-08-25
rfc: "0113-branch-and-guard-parity"
cluster: invented-arm
packages: []
deps: []
deps-rfc: []
est-loc: 30
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

# Delete the callerless `warningMessage(origin, baseUrl)` free function

## Context

Surfaced while reviewing PR #6703.

`packages/actionpack/src/action-controller/metal/request-forgery-protection.ts:176`
exports a free function with no Rails counterpart:

    export function warningMessage(origin?: string | null, baseUrl?: string | null): string {
      if (origin && baseUrl && origin !== baseUrl) {
        return `HTTP Origin header (${origin}) didn't match request.base_url (${baseUrl})`;
      }
      return "Can't verify CSRF token authenticity.";
    }

It duplicates both message strings of Rails'
`unverified_request_warning_message`
(`actionpack/lib/action_controller/metal/request_forgery_protection.rb:411-417`):

    def unverified_request_warning_message
      if valid_request_origin?
        "Can't verify CSRF token authenticity."
      else
        "HTTP Origin header (#{request.origin}) didn't match request.base_url (#{request.base_url})"
      end
    end

which is already ported in the same file as `unverifiedRequestWarningMessage`
and is now reached on the real request path through `handleUnverifiedRequest`
(PR #6703). The free function has **no callers** anywhere in the repo — `grep -rn
"warningMessage"` under `packages/actionpack/src` finds only its own
definition, the `"warningMessage" in protectionStrategy` check inside
`handleUnverifiedRequest`, and the `Exception#warningMessage` field.

It also inverts the Rails branch: Rails keys on `valid_request_origin?`, which
returns true when `origin` is nil (rb:627-635), whereas this keys on
`origin && baseUrl && origin !== baseUrl` — a bare JS truthiness test, so an
empty-string origin takes the wrong arm.

`parity:api:extra` does not flag it because the name collides with Rails'
`ProtectionMethods::Exception#warning_message` accessor (rb:307), which is a
real Ruby member — so the extra surface is invisible to the gate and only
shows on a read.

## Converged shape

Delete the function and its export. `unverifiedRequestWarningMessage` is the
ported seat for both strings; nothing needs to replace it.

## Acceptance criteria

- [ ] `warningMessage(origin, baseUrl)` is deleted from
      `request-forgery-protection.ts` and from any barrel that re-exports it.
- [ ] `unverifiedRequestWarningMessage` remains the only source of the two
      message strings.
- [ ] actionpack tests pass; `pnpm parity:api` delta non-negative.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0113-branch-and-guard-parity`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
