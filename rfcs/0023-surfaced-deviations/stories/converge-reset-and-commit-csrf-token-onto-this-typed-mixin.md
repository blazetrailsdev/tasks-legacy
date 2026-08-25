---
title: "converge-reset-and-commit-csrf-token-onto-this-typed-mixin"
status: ready
updated: 2026-08-18
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "actionpack"
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

# Converge `reset_csrf_token` / `commit_csrf_token` onto the `this`-typed mixin

## Context

Split out of `0108-call-gate-false-positives/request-forgery-protection-this-typed-mixin`
(PR #6703), which moved the other 22 members of
`packages/actionpack/src/action-controller/metal/request-forgery-protection.ts`
onto CLAUDE.md's `this`-typed mixin idiom and left this pair alone because
converging it is a body change, not a receiver change.

Rails (`actionpack/lib/action_controller/metal/request_forgery_protection.rb:371-379`):

    def reset_csrf_token(request) # :doc:
      request.delete_header(CSRF_TOKEN)
      csrf_token_storage_strategy.reset(request)
    end

    def commit_csrf_token(request) # :doc:
      csrf_token = request.env[CSRF_TOKEN]
      csrf_token_storage_strategy.store(request, csrf_token) unless csrf_token.nil?
    end

Both are instance methods: the storage strategy comes off implicit `self`, and
`request` is the argument. trails
(`request-forgery-protection.ts:184-190`) instead takes the strategy as an
explicit first parameter and the storage as the second, and neither body
touches `request.env[CSRF_TOKEN]` at all:

    export function resetCsrfToken<T>(csrfStore: CsrfTokenStore<T>, storage: T): void {
      csrfStore.reset(storage);
    }

    export function commitCsrfToken<T>(csrfStore: CsrfTokenStore<T>, storage: T, token: string): void {
      csrfStore.store(storage, token);
    }

So three things diverge, not one: the receiver, the parameter list, and the
`CSRF_TOKEN` env read/delete that is half of what each method does. The
`CSRF_TOKEN` constant is already in the file as `CSRF_TOKEN_ENV_KEY`
(`request-forgery-protection.ts`, mirroring rb:64) and `realCsrfToken` already
reads through it, so the env half has a seat to land in.

There are no call sites outside the module today (`grep -rn "resetCsrfToken\|commitCsrfToken"`
finds only the definitions and the barrel export), so the signature change is
free.

## Acceptance criteria

- [ ] `resetCsrfToken` / `commitCsrfToken` take `this: CsrfController` and a
      `request` argument, reading the strategy off `this.csrfTokenStorageStrategy`
      as rb:371-379 reads it off `self`.
- [ ] `reset_csrf_token` deletes the `CSRF_TOKEN` header and
      `commit_csrf_token` reads `request.env[CSRF_TOKEN]` and stores only when it
      is not nil — Ruby `unless csrf_token.nil?`, so a stored `false`/`""` still
      commits.
- [ ] The `reset_csrf_token → delete` baseline row in
      `call-mismatches-exclude/actioncontroller/metal/request-forgery-protection.json`
      is deleted by hand (only-shrink, no reseed) and the mark tightened with
      `pnpm parity:api:calls:tighten`.
- [ ] `pnpm parity:api:calls` / `:args` green; actionpack tests pass.

_Moved from RFC 0108 on 2026-08-18, which is closed. Two reasons, both of which
0108's closing note names explicitly: this is **port convergence** (a body
change, as the context above says outright) rather than a call-gate false
positive, and it is **actionpack**, outside 0108's declared package list. It
lands in 0023 per CLAUDE.md — a surfaced deviation with no better-fit active
RFC; 0106 is the convergence home only for activerecord/arel/activesupport._
