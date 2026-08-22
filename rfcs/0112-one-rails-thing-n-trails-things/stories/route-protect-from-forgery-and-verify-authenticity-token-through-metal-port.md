---
title: "Route protect_from_forgery / verify_authenticity_token through the metal CSRF port"
status: ready
updated: 2026-08-22
rfc: "0112-one-rails-thing-n-trails-things"
cluster: dead-mixin-companions
packages: []
deps: []
deps-rfc: []
est-loc: 400
priority: 30
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

# Route `protect_from_forgery` / `verify_authenticity_token` through the metal CSRF port

## Context

Surfaced while reviewing PR #6703, which moved
`packages/actionpack/src/action-controller/metal/request-forgery-protection.ts`
onto the `this`-typed mixin idiom and ported `handle_unverified_request`.

`ActionController::Base`'s two CSRF entry points bypass that metal port
entirely and drive a trails-invented class instead —
`RequestForgeryProtection` in
`packages/actionpack/src/action-dispatch/request-forgery-protection.ts`, which
has no Rails counterpart (Rails has no such class; the module IS the
implementation).

Rails (`actionpack/lib/action_controller/metal/request_forgery_protection.rb:199-208`):

    def protect_from_forgery(options = {})
      options = options.reverse_merge(prepend: false)

      self.forgery_protection_strategy = protection_method_class(options[:with] || :null_session)
      self.request_forgery_protection_token ||= :authenticity_token

      self.csrf_token_storage_strategy = storage_strategy(options[:store] || SessionStore.new)

      before_action :verify_authenticity_token, options
      append_after_action :verify_same_origin_request
    end

and (`:391-398`):

    def verify_authenticity_token # :doc:
      mark_for_same_origin_verification!

      if !verified_request?
        logger.warn unverified_request_warning_message if logger && log_warning_on_csrf_failure

        handle_unverified_request
      end
    end

trails (`packages/actionpack/src/action-controller/base.ts:434-464`):

    static protectFromForgery(options: { with?: ... } = {}): void {
      this._csrfProtection = new RequestForgeryProtection({ strategy: options.with ?? "exception" });
    }

    verifyAuthenticityToken(): void {
      const csrf = (this.constructor as typeof Base)._csrfProtection;
      if (!csrf) return;
      const token = (this.params.get("authenticity_token") as string) ?? this.request?.getHeader("x-csrf-token") ?? null;
      const result = csrf.verifyRequest({ method: ..., session: ..., token, host: ... });
      if (!result.verified) csrf.handleUnverified(this.session);
    }

Six divergences, all in these two bodies:

- `protect_from_forgery` defaults `with:` to `:null_session`; trails defaults to
  `"exception"`.
- It sets three class attributes (`forgery_protection_strategy`,
  `request_forgery_protection_token ||=`, `csrf_token_storage_strategy` from
  `options[:store]`); trails sets one private `_csrfProtection` field and drops
  `store:` entirely.
- It registers `before_action :verify_authenticity_token` and
  `append_after_action :verify_same_origin_request`; trails registers neither,
  so `verify_same_origin_request` (rb:429-437) never runs.
- `verify_authenticity_token` calls `mark_for_same_origin_verification!` first;
  trails never calls it, so `marked_for_same_origin_verification?` is always
  false and the cross-origin-JavaScript guard is dead.
- It logs `unverified_request_warning_message` under
  `logger && log_warning_on_csrf_failure`; trails logs nothing.
- It dispatches through `handle_unverified_request`, which is what selects the
  strategy and sets `warning_message`; trails calls `csrf.handleUnverified`.

Every callee is now ported in the metal file and takes an implicit receiver:
`markForSameOriginVerificationBang`, `isVerifiedRequest`,
`unverifiedRequestWarningMessage`, `handleUnverifiedRequest`,
`verifySameOriginRequest`, `protectionMethodClass`, `storageStrategy`.

## Converged shape

- `protectFromForgery` becomes the rb:199-208 body, assigning
  `forgeryProtectionStrategy` / `requestForgeryProtectionToken` /
  `csrfTokenStorageStrategy` on the class and registering the before/after
  actions. It belongs in the metal file at the Rails name, assigned onto the
  class per the CLAUDE.md mixin convention, not in `base.ts`.
- `verifyAuthenticityToken` becomes the rb:391-398 body, calling the four metal
  functions in Rails' order with Rails' guards.
- `packages/actionpack/src/action-dispatch/request-forgery-protection.ts`'s
  `RequestForgeryProtection` class loses its callers and can then be deleted —
  it is a trails invention with no Rails counterpart.

Note `packages/actionpack/src/action-controller/controller/request-forgery-protection.test.ts`
asserts against that invented class throughout, so most of that file has to be
re-pointed at the metal port as part of this. Rails' own
`RequestForgeryProtectionTests` module is the shape to port toward.

## Acceptance criteria

- [ ] `protect_from_forgery` and `verify_authenticity_token` mirror rb:199-208
      and rb:391-398 — same assignments, same callees, same order, same
      `:null_session` default.
- [ ] `verify_same_origin_request` is registered as an after-action and
      `mark_for_same_origin_verification!` runs, so the cross-origin-JavaScript
      guard is live.
- [ ] `RequestForgeryProtection` (action-dispatch) is deleted, or the remaining
      callers are named and filed.
- [ ] `pnpm parity:api:calls` / `:args` green; no new baseline rows.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
