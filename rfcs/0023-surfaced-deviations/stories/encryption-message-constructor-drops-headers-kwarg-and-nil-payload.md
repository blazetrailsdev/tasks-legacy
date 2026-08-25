---
title: "Message constructor drops Rails' headers: kwarg and coerces a nil payload to empty string"
status: draft
updated: 2026-07-30
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 90
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Rails' `ActiveRecord::Encryption::Message#initialize` is

```ruby
def initialize(payload: nil, headers: {})
  validate_payload_type(payload)

  @payload = payload
  @headers = Properties.new(headers)
end
```

(`vendor/rails/activerecord/lib/active_record/encryption/message.rb:14-19`).

trails' constructor (`packages/activerecord/src/encryption/message.ts:14`) is
`constructor(payload?: string | Buffer | null)` — two deviations:

1. **No `headers:` argument.** Every Rails test that builds a message with
   headers uses it, e.g.
   `Message.new(payload: "some other secret payload", headers: { some_header: "some other value" })`
   (`activerecord/test/cases/encryption/message_serializer_test.rb:19`,
   `message_pack_message_serializer_test.rb:20`, `:64`). The ported tests have to
   construct and then call `addHeader`/`addHeaders` instead — and those two
   methods are trails inventions with no Rails counterpart (they are the
   `encryption/message.ts — 1 novel` entry in `parity:api:extra`). Rails' only writer is
   `headers[key] = value` through `Properties#[]=`.

2. **A nil payload becomes `""`.** `this.payload = payload ?? ""` where Rails
   keeps `@payload = payload`, i.e. nil. `Message.new.payload` is `nil` in Rails
   and `""` in trails. This is now observable through `Message#==` (ported in
   PR #5630, `message.rb:21`): a nil-payload message and an empty-string one
   compare equal in trails and unequal in Rails.

`Properties.new(initial)` already accepts the hash, so wiring the kwarg through
is mostly signature work plus updating call sites.

## Acceptance criteria

- [ ] `Message`'s constructor takes Rails' two named arguments (`payload`,
      `headers`) and passes `headers` to `Properties`.
- [ ] A nil/absent payload stays nil rather than becoming `""`, and
      `Message#equals` distinguishes the two (Rails: `nil != ""`).
- [ ] `addHeader` / `addHeaders` are retired in favour of the `headers:` kwarg
      and `headers.set` (Rails' `headers[key] =`), removing the novel
      extra-surface entry for `encryption/message.ts` — or the retention is
      justified at the call site.
- [ ] Ported tests that build messages with headers use the kwarg, matching the
      Rails test bodies they are named after.
- [ ] `parity:api` / `parity:api:extra` / `parity:test` deltas stay non-negative.
