---
title: "call-define-attribute-methods-from-init-internals"
status: claimed
updated: 2026-08-23
rfc: "0115-activemodel-fidelity-convergence"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: null
claim: "2026-08-23T19:38:27Z"
assignee: "call-define-attribute-methods-from-init-internals"
blocked-by: null
closed-reason: null
---

# Call `define_attribute_methods` from `Core#init_internals`

## Context

Split out of RFC 0115 `port-core-init-internals-missing-assignments` after a
failed port attempt on trails#6933 (three red CI lanes). Rails' last line of
`init_internals` is
`klass.define_attribute_methods`
(`vendor/rails/activerecord/lib/active_record/core.rb:849`) — the call that
makes a class's attribute methods exist by construction time rather than on
first `method_missing`.

trails omits it and generates at the end of every schema load instead
(`defineAttributeMethodsAfterLoad`, `packages/activerecord/src/model-schema.ts`).

**The blocker is not the call — it is a duplicate generated-module carrier.**
Adding `klass.defineAttributeMethods()` to `initInternals`
(`packages/activerecord/src/core.ts`) makes a record constructed before its
schema loads generate onto one carrier; `applyColumnsHash`
(`model-schema.ts:1301-1302`) then clears `_attributeMethodsGenerated` so the
next demand regenerates, and the post-load pass seats a SECOND carrier holding
the same names. Probed on a `class Employee extends Base` with
`attribute("name", "string")`: walking `Employee.prototype`'s chain for
`nameChanged` finds two owning links with the call in place and one without it.

`undefineAttributeMethods` (`attribute-methods.ts:523-528`) clears only one of
the two, so the accessor survives an undefine. That reds
`attribute-methods.trails.test.ts:397` on SQLite, PostgreSQL and MariaDB
alike, and it is behaviour Rails' own
`attribute_methods_test.rb:1098-1117` depends on — `undefine_attribute_methods`
must leave `method_defined?` false until something regenerates.

Note for whoever picks this up: a `@missingRailsCall` tag is NOT available as an
interim record here. `parity:api:calls` does not flag this omission, so a tag on
it fails the gate as a STALE tag ("1 STALE @missingRailsCall tag(s) whose call
is no longer flagged"). The reason lives in prose at the call site until the
call actually lands.

## Acceptance criteria

- [ ] Generation seats exactly one generated-module carrier per class across a
      pre-schema construction plus a post-load regeneration (covered by a test
      that fails on today's code with the call in place).
- [ ] `Core#initInternals` calls `klass.defineAttributeMethods()` in
      core.rb:849's position, and the prose note in its JSDoc is deleted.
- [ ] `undefineAttributeMethods` clears the accessor for a class that was
      constructed before its schema loaded.
- [ ] `attribute-methods.test.ts`, `attribute-methods.trails.test.ts` and
      `base.test.ts` stay green on SQLite, PostgreSQL and MySQL/MariaDB.
