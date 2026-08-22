---
title: "memoize EncryptedAttributeType#clean_text_scheme"
status: ready
updated: 2026-08-22
rfc: "0112-one-rails-thing-n-trails-things"
cluster: split-stores
packages: []
deps: []
deps-rfc: []
est-loc: 20
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Rails memoizes the clean-text scheme:
`activerecord/lib/active_record/encryption/encrypted_attribute_type.rb:162-163`

```ruby
def clean_text_scheme
  @clean_text_scheme ||= ActiveRecord::Encryption::Scheme.new(downcase: downcase?, encryptor: ActiveRecord::Encryption::NullEncryptor.new)
end
```

trails' `cleanTextScheme`
(`packages/activerecord/src/encryption/encrypted-attribute-type.ts:303-308`)
builds a fresh `Scheme` (and a fresh `NullEncryptor`) on every call. Its only
caller is `previousSchemes` (encrypted_attribute_type.rb:67), which is itself
consulted per read on a `supportUnencryptedData` attribute, so the missing memo
is both a divergence and per-row allocation.

The rest of that file already memoizes the Rails way (`_previousTypes`,
`_previousTypesWithoutCleanText`); this one method was missed.

## Acceptance criteria

- [ ] `cleanTextScheme` memoizes into a `_cleanTextScheme` field, mirroring
      `@clean_text_scheme ||=` (encrypted_attribute_type.rb:162-163).
- [ ] Repeated calls return the identical `Scheme` instance.
- [ ] `pnpm vitest run packages/activerecord/src/encryption` stays green.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
