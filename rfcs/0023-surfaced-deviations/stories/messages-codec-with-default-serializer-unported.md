---
title: "Messages::Codec.with(default_serializer:) unported; encrypted-file test reduced to a round-trip"
status: draft
updated: 2026-08-17
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activesupport"
deps: []
deps-rfc: []
est-loc: 140
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`ActiveSupport::Messages::Codec.with(default_serializer: …)` is unported, so the
`encrypted_file_test.rb` test named for it cannot assert what its name promises.

Rails (`vendor/rails/activesupport/test/encrypted_file_test.rb:148-156`):

```ruby
test "can read encrypted file after changing default_serializer" do
  ActiveSupport::Messages::Codec.with(default_serializer: :marshal) do
    encrypted_file(@content_path).write(@content)
  end

  ActiveSupport::Messages::Codec.with(default_serializer: :json) do
    assert_equal @content, encrypted_file(@content_path).read
  end
end
```

The point is that a payload written under one global default serializer still
decodes after the default flips — the envelope carries its own format marker.
Ours (`packages/activesupport/src/encrypted-file.test.ts`, final test) reduces to
a plain write/read round-trip, documented inline, because there is no global
flip-able serializer state: `EncryptedFile` hardcodes its serializer and
`MessageEncryptor` takes one per instance.

Related but distinct: `encrypted-file-serializer-marshal-not-nullserializer`
covers _which_ serializer `EncryptedFile#encryptor` passes
(`encrypted_file.rb:113`). This story covers the missing `Codec.with` seat and
the test that depends on it; converging the serializer alone does not restore
the scenario.

## Converged shape

Port `Messages::Codec`'s `default_serializer` seat and its block-scoped `with`
override (`vendor/rails/activesupport/lib/active_support/messages/codec.rb`), then
restore the test to Rails' two-block shape so the cross-serializer read is
actually exercised.

## Acceptance criteria

- `Messages::Codec.with(default_serializer: …)` exists and scopes the default to
  the block, restoring the prior value on exit and on throw.
- "can read encrypted file after changing default_serializer" ports as Rails
  writes it — write under one serializer, read under another, one `assert_equal`.
- The inline comment explaining the reduction is deleted.
- `encrypted_file_test.rb` stays at 0 assertion mismatches.
