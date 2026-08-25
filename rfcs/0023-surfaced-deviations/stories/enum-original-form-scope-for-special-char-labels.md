---
title: "Enum: original-form scopes for special-char labels (Rails literal-label scope surface)"
status: ready
updated: 2026-07-27
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 40
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Rails defines every enum value's scope under the literal label —
`define_enum_methods` (vendor/rails/activerecord/lib/active_record/enum.rb:268)
generates `scope value_method_name` even for special-char labels like
`"Etc/GMT+1"`, so `klass.public_send(:"Etc/GMT+1")` works
(enum_test.rb "mangling collision for enum names", ~:951).

Trails' `_enum` (packages/activerecord/src/enum.ts:810-834) routes scope names
through `camelize`, which maps `/` → `::` — the scope for label "Etc/GMT+1"
lands as `etc::GMT+1`. The original-form surface for special-char labels
(enum.ts:836-856) defines only the predicate (`isEtc/GMT+1`) and bang
(`Etc/GMT+1Bang`) — no original-form scope. PR #5087's ported
"mangling collision for enum names" test therefore calls the scope via
`["etc::GMT+1"]` with a call-site deviation comment
(packages/activerecord/src/enum.test.ts, mangling test).

## Acceptance criteria

- [ ] Special-char labels get an original-form class-method scope (bracket
      notation, e.g. `Klass["Etc/GMT+1"]()`), mirroring Rails' literal
      define_method scope surface, alongside the existing camelized scope.
- [ ] The mangling test's scope call switches to the original label form and
      its deviation comment is removed.
