---
title: "CallTemplate::ProcCall keeps Rails' @override_target name and its || target fallback"
status: ready
updated: 2026-08-25
rfc: "0113-branch-and-guard-parity"
cluster: missing-arm
packages: []
deps: []
deps-rfc: []
est-loc: 80
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

# `CallTemplate::ProcCall` keeps Rails' `@override_target` name and its `|| target` fallback

## Context

Surfaced in PR #6585 while extracting `CallTemplate.build`
(`vendor/rails/activesupport/lib/active_support/callbacks.rb:494-512`). The
factory now dispatches to `ProcCall` for a `Conditionals::Value` filter exactly
as Rails does, but `ProcCall` itself still diverges — pre-existing, untouched by
that PR beyond widening its constructor type.

Rails (`callbacks.rb`, `class ProcCall`):

    def initialize(target)
      @override_target = target
    end

    def expand(target, value, block)
      [@override_target || target, block, :call, target, value]
    end

    def make_lambda
      lambda do |target, value, &block|
        (@override_target || target).call(target, value, &block)
      end
    end

trails (`packages/activesupport/src/callbacks.ts`, `class ProcCall`):

- the field is named `fn`, not `overrideTarget`. CLAUDE.md: a field keeps the
  Rails identifier, camelCased.
- every method drops the `|| target` fallback and calls the stored value
  unconditionally, so a `ProcCall` constructed with a nil/absent target — which
  Rails resolves to the runtime `target` — has no arm here at all.

## Converged shape

Field renamed `overrideTarget`, and `expand` / `makeLambda` / `invertedLambda`
each restore the `(this.overrideTarget ?? target)` fallback with Ruby
truthiness semantics (`!= null`, per CLAUDE.md — not a bare truthiness test).

The `typeof f === "function"` discrimination introduced in #6585 stays: Ruby
reaches both a Proc and a `Conditionals::Value` through one `.call`, and a JS
function's own `.call` has different semantics, so the union must be
discriminated. That part is language-forced and already cited at the call site.

`make` is a trails-only helper on this class and should be checked for a Rails
counterpart while in there.

## Acceptance criteria

- [ ] `ProcCall`'s field is `overrideTarget`.
- [ ] `expand`, `makeLambda` and `invertedLambda` carry the `|| target` fallback.
- [ ] `pnpm parity:api:calls` / `pnpm parity:api:calls:args` green; no new
      `pnpm parity:api:extra` surface.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0113-branch-and-guard-parity`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
