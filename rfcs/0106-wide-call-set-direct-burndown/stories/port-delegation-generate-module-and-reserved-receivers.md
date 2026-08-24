---
title: "port-delegation-generate-module-and-reserved-receivers"
status: in-progress
updated: 2026-08-24
rfc: "0106-wide-call-set-direct-burndown"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: 4
pr: 6995
claim: "2026-08-24T15:57:23Z"
assignee: "port-delegation-generate-module-and-reserved-receivers"
blocked-by: null
closed-reason: null
---

# Port `Delegation.generate`'s `to:`-a-Module branch and reserved-receiver handling

## Context

Split out of `module-ext-delegate-should-call-delegation-generate` (PR #6863),
which converged `Module#delegate` onto `ActiveSupport::Delegation.generate` but
left two arms of `generate` itself unported. Neither is a regression — trails'
`DelegateOptions.to` has always been a bare `string` — but the parent story's
"ported method by method against the Ruby" is only partly met until they land.

`vendor/rails/activesupport/lib/active_support/delegation.rb:36-58`:

    receiver = if to.is_a?(Module)
      if to.name.nil?
        raise ArgumentError, "Can't delegate to anonymous class or module: #{to}"
      end

      unless Inflector.safe_constantize(to.name).equal?(to)
        raise ArgumentError, "Can't delegate to detached class or module: #{to.name}"
      end

      "::#{to.name}"
    else
      to.to_s
    end
    receiver = "self.#{receiver}" if RESERVED_METHOD_NAMES.include?(receiver)

Two missing pieces:

1. **`to:` may be a Module.** `delegate :sum, to: MyModule` delegates to the
   constant, with the two `ArgumentError`s above guarding an anonymous or
   detached one. trails types `to` as `string` only, so this call shape does not
   type-check, let alone run. `RESERVED_METHOD_NAMES` (`delegation.rb:15-17`) is
   the Ruby keyword set plus `_`/`arg`/`args`/`block`.
2. **A reserved receiver is prefixed with `self.`**, so `delegate :x, to: :class`
   generates `self.class.x`. trails reads `this[to]`, which reaches the same
   value for a method-named target but has no `self.` disambiguation step and no
   set to check against.

`generate`'s JSDoc in `packages/activesupport/src/delegation.ts` points at this
story; delete that pointer when it lands.

## Acceptance criteria

- [ ] `DelegateOptions.to` accepts the trails spelling of a Ruby Module as well
      as a method name, with both `ArgumentError`s at `delegation.rb:38, 42` —
      same class, same message string.
- [ ] `RESERVED_METHOD_NAMES` is ported at the Rails name and consulted where
      Rails consults it (`delegation.rb:58`).
- [ ] The JSDoc pointer in `Delegation.generate` is removed, not reworded.
- [ ] `pnpm parity:api:calls` / `:args` green; `pnpm parity:api:extra --package
activesupport` shows no new untagged surface.
