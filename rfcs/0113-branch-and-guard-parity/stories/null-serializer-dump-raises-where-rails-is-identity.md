---
title: "NullSerializer.dump raises TypeError where Rails' is identity"
status: ready
updated: 2026-08-25
rfc: "0113-branch-and-guard-parity"
cluster: invented-arm
packages: []
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

`MessageEncryptor::NullSerializer`
(`vendor/rails/activesupport/lib/active_support/message_encryptor.rb:105-113`)
is two identity methods with no type check:

```ruby
module NullSerializer # :nodoc:
  def self.load(value)
    value
  end

  def self.dump(value)
    value
  end
end
```

trails (`packages/activesupport/src/message-encryptor.ts`, bottom of file) adds
a raise Rails does not have:

```ts
export function dump(value: unknown): string {
  if (typeof value !== "string") {
    throw new TypeError("NullSerializer.dump expects a string value");
  }
  return value;
}
```

That is an invented raise site with an invented message. It is currently
unreachable on the encryptor's own path (PR #6219 routed `sign` through a
`MessageVerifier` built with this serializer, and the value is always the
encoded ciphertext string), but `NullSerializer` is exported and any caller
handing it a non-String gets a `TypeError` where Rails hands the value straight
back.

Rails also declares `NullSerializer` right after `default_cipher`
(`message_encryptor.rb:105`), before `InvalidMessage` and the constants; trails
declares it at the very bottom of the file, after the class body. Worth fixing
in the same pass so `rails-file-structure-method-order` sees Rails' layout.

## Converged shape

```ts
export namespace NullSerializer {
  export function load(value: string): string {
    return value;
  }
  export function dump(value: unknown): unknown {
    return value;
  }
}
```

declared where `message_encryptor.rb:105` declares it. The `TypeError` and its
message are deleted, not rehomed.

## Acceptance criteria

- [ ] `NullSerializer.dump` and `.load` are identity, with no type check.
- [ ] `NullSerializer` is declared in Rails' position in the file.
- [ ] `message-encryptor.test.ts` and `messages/*` suites pass unchanged.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0113-branch-and-guard-parity`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
