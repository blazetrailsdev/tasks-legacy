---
title: "Port Delegation.generate_method_missing's reserved / __target receiver prefix"
status: in-progress
updated: 2026-08-24
rfc: "0106-wide-call-set-direct-burndown"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 90
priority: null
pr: 6998
claim: "2026-08-24T18:07:08Z"
assignee: "stale-story-references-scan-times-out-under-load"
blocked-by: null
closed-reason: null
---

# Port `Delegation.generate_method_missing`'s reserved / `__target` receiver prefix

## Context

Split out of `port-delegation-generate-module-and-reserved-receivers` (PR #6995),
which ported `RESERVED_METHOD_NAMES` and the `self.` receiver prefix into
`Delegation.generate` only. Rails applies the same shaping in
`generate_method_missing`, with one extra name:

`vendor/rails/activesupport/lib/active_support/delegation.rb:160-162`:

    def generate_method_missing(owner, target, allow_nil: nil)
      target = target.to_s
      target = "self.#{target}" if RESERVED_METHOD_NAMES.include?(target) || target == "__target"

trails' `generateMethodMissing`
(`packages/activesupport/src/delegation.ts`) takes `target` as a bare string and
reads `obj[target]` in both the `has` and `get` traps with no shaping step, so a
reserved target and a target literally named `__target` — which Rails must
disambiguate from the local the generated body binds (`:164`, `:185`) — are
indistinguishable from any other name.

## Converged shape

Consult the `RESERVED_METHOD_NAMES` set that PR #6995 already exported from the
`Delegation` namespace, plus the `target == "__target"` arm, at
`delegation.rb:162`, and strip the `self.` back off for the member read the same
way `generate` does — the prefix is a Ruby parser disambiguation with no JS
member-read effect, but it is observable in the `DelegationError` message
(`nil_target(name, :'#{target}')`, `:176`, `:196`).

## Acceptance criteria

- [ ] `RESERVED_METHOD_NAMES.include?(target) || target == "__target"` ported at
      `delegation.rb:162`, same order, same set.
- [ ] The `DelegationError` raised from both traps carries the prefixed target,
      matching `:176` / `:196`.
- [ ] `pnpm parity:api:calls` / `:args` green; no new extra surface.
