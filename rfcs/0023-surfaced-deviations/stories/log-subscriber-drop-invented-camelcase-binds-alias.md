---
title: "Drop log-subscriber's invented camelCase typeCastedBinds payload alias"
status: ready
updated: 2026-07-27
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 5
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while auditing the `type_casted_binds` payload consumers for PR #4866.

`packages/activerecord/src/log-subscriber.ts:112-113` reads the payload slot as:

```ts
const castedParams = this.typeCastedBinds(payload.type_casted_binds ?? payload.typeCastedBinds);
```

Rails' LogSubscriber (`vendor/rails/activerecord/lib/active_record/log_subscriber.rb:33`)
reads exactly one key — `payload[:type_casted_binds]`. The `?? payload.typeCastedBinds`
camelCase fallback is a trails invention with no Rails counterpart.

It is also dead. A full audit of payload producers
(`git grep -n "typeCastedBinds:" -- packages/activerecord/src --include=*.ts`)
returns only function PARAMETER declarations —
`abstract-adapter.ts:2337`, `abstract/database-statements.ts:1875`,
`mysql2/database-statements.ts:228`, `postgresql/database-statements.ts:128`,
`sqlite3/database-statements.ts:166` — never a payload key. Every producer emits
the Rails-named snake_case `type_casted_binds:` (16 sites, inventoried in
`type-casted-binds-payload-self-dispatch`). So the fallback can never fire.

Low value on its own, but it is an invented alias sitting in a Rails-ported file,
and it invites new producers to emit a camelCase key that Rails has no name for —
which would then diverge silently from every other subscriber.

## Acceptance criteria

- [ ] `log-subscriber.ts` reads only `payload.type_casted_binds`, matching
      `log_subscriber.rb:33`.
- [ ] Confirm no producer emits a `typeCastedBinds` payload key before removing
      (re-run the grep above; parameter names are not producers).
- [ ] `log-subscriber.test.ts` / `log-subscriber.trails.test.ts` still pass —
      they construct payloads with the snake_case key.
- [ ] The callable-unwrap in `typeCastedBinds` (`:197-200`) is UNCHANGED — it
      mirrors `log_subscriber.rb:61-63`'s `respond_to?(:call)` and is load-bearing
      for the lazy slot shipped in #4866.
