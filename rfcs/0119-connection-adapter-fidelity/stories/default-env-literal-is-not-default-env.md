---
title: 'defaultEnv terminal literal is "default"/"development", not Rails'' "default_env"'
status: draft
updated: 2026-07-28
rfc: "0119-connection-adapter-fidelity"
cluster: null
deps: []
deps-rfc: []
est-loc: 60
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Rails' terminal env fallback is the literal string `"default_env"`:
`DEFAULT_ENV = -> { RAILS_ENV.call || "default_env" }`
(`vendor/rails/activerecord/lib/active_record/connection_handling.rb:7`).

trails uses two different literals instead, neither of which is `"default_env"`
(`packages/activerecord/src/database-configurations.ts`):

- `DatabaseConfigurations.defaultEnv` returns `"default"` when `_defaultEnv` is
  set to a blank string.
- It returns `"development"` when `_defaultEnv` is unset, which is also
  `currentEnv()`'s terminal fallback.

The divergence is enshrined in a ported test: `connection-handler.test.ts`'s
"default env fall back to default env when rails env or rack env is empty
string" asserts `"default"`, whereas the Rails original
(`vendor/rails/activerecord/test/cases/connection_adapters/connection_handler_test.rb:27`)
asserts `assert_equal "default_env", ActiveRecord::ConnectionHandling::DEFAULT_ENV.call`.
So the ported test currently encodes the trails literal rather than Rails'.

Noticed while shipping PR #5496, which touched the surrounding precedence chain
but deliberately left both literals' observable behaviour unchanged to keep the
diff scoped.

## Acceptance criteria

- [ ] Establish whether `"default_env"` is the correct terminal fallback for
      both the blank and unset cases, or whether `"development"` is a
      deliberate trails default worth keeping (and if so, cite it).
- [ ] If converging: the `defaultEnv` getter returns Rails' literal, and the
      ported test asserts the same value the Rails original does.
- [ ] Sweep callers that rely on the current `"development"` fallback
      (constructor build env, `fromRaw`, `fromEnv`, `schema.ts`,
      `migration.ts`, `tasks/database-tasks.ts`) before flipping it.
- [ ] No test names change.
