---
title: "generated_association_methods returns a name Set where Rails includes a Module"
status: draft
updated: 2026-08-20
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 200
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

# `generated_association_methods` returns a name Set where Rails builds and includes a Module

## Context

Surfaced converging `core.json`'s call-set rows in PR #6762 (RFC 0106 wave 4c).
The `generated_association_methods -> include` row could not be converged there
and now carries a per-site reason; this story is that row's burndown.

Rails (`vendor/rails/activerecord/lib/active_record/core.rb:338-346`):

    def generated_association_methods # :nodoc:
      @generated_association_methods ||= begin
        mod = const_set(:GeneratedAssociationMethods, Module.new)
        private_constant :GeneratedAssociationMethods
        include mod

        mod
      end
    end

Three things happen: a real anonymous Module is created and named under the
class, it is made a private constant, and it is **included** — so generated
association methods land in an ancestor of the model rather than on the model
itself, which is what lets a hand-written method in the class body override a
generated one by ordinary method lookup.

`packages/activerecord/src/core.ts:713-718` returns a `Set<string>` of generated
method names instead, and `initialize_generated_modules` (core.rb:338) just
calls it. Nothing is included, so association methods are defined directly on
the class and the override ordering Rails gets for free has to be re-derived by
whatever defines them.

Note the repo already has the settled shapes for this: `include()` /
`Included<>` from `@blazetrails/activesupport` (see CLAUDE.md "Module mixins"),
and the sibling story `define-dirty-attribute-methods-into-generated-module`
(0023) is the same convergence on the dirty-tracking side — coordinate with it
rather than inventing a second mechanism.

## Converged shape

`generatedAssociationMethods` memoizes and returns a real module object that is
`include()`d into the class on first call, with association method generation
targeting that module. The `Set<string>` becomes an implementation detail of the
module (or disappears), and the `generated_association_methods -> include` row
is deleted from
`scripts/api-compare/call-mismatches-exclude/activerecord/core.json` by hand via
`serializeBaseline`, then the shard mark tightened.

## Acceptance criteria

- [ ] `generatedAssociationMethods` mirrors core.rb:338-346, including the
      `include`.
- [ ] Generated association methods live in the included module, and a class-body
      method of the same name still wins.
- [ ] The `include` row is deleted from `core.json`; mark tightened. No reseed.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.
