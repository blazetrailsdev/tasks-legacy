---
title: "has_secure_password stores the plaintext in an ivar pair, not an attribute plus an after_initialize eviction"
status: ready
updated: 2026-08-23
rfc: "0115-activemodel-fidelity-convergence"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 220
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`packages/activemodel/src/secure-password.ts:109` registers an
`after_initialize` callback that reads the plaintext password out of the
attribute set, deletes the attribute, and re-runs the writer:

```ts
modelClass.setCallback("initialize", "after", (record: Model) => {
  if (record._attributes.has(attribute)) {
    const plaintext = record._readAttribute(attribute);
    record._attributes.delete(attribute);
    setPassword(record, plaintext, attribute, digestAttr, passwordCache);
  }
});
```

**Rails' `has_secure_password` registers no callback at all.** The plaintext
never becomes an attribute in the first place:
`activemodel/lib/active_model/secure_password.rb:184` is `attr_reader attribute`
and `:186-192` is `define_method("#{attribute}=")`, whose body writes the
plaintext to an **ivar** — `instance_variable_set("@#{attribute}", …)` at `:188`
and `:191` — alongside the digest. Because the reader and writer are a plain
ivar pair, there is no attribute-set entry to clean up, and so no lifecycle hook
to clean it up with.

The trails version instead lets the plaintext land in `_attributes` and then
evicts it one hook later. That is an invented lifecycle hook standing in for a
storage decision, and it is why this file needed touching at all when PR #6923
retired the `afterInitialize` macro (it was respelled `setCallback("initialize",
"after", …)` there, which converged the _spelling_ but not the hook).

Note the callback was also the reason `respond_to?("define_method_attribute")`
pressure reaches this file; converging the storage removes that coupling.

## Converged shape

Follow `secure_password.rb:184-192`: the generated `password` reader and
`password=` writer are an ivar pair (a private per-instance slot in TS — the
`passwordCache` WeakMap already in this file is the natural seat), the digest is
written in the writer, and the `after_initialize` registration is deleted.

## Acceptance criteria

- No `initialize` callback is registered by `hasSecurePassword`.
- The plaintext never enters `_attributes`; `password` / `password=` are the
  generated reader/writer pair of `secure_password.rb:184-192`.
- `packages/activemodel/src/secure-password.test.ts` stays green with no test
  renamed.
- Parity deltas non-negative; `pnpm parity:api:calls` / `:args` clean, no reseed.
