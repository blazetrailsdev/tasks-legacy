---
title: "One Rails encrypts, two trails implementations"
status: ready
updated: 2026-08-22
rfc: "0112-one-rails-thing-n-trails-things"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 150
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

# One Rails `encrypts`, two trails implementations

## Context

`encryptable_record.rb:49-55` defines exactly one `encrypts`:

```ruby
def encrypts(*names, key_provider: nil, key: nil, deterministic: false, ...)
  self.encrypted_attributes ||= Set.new # not using :default because the instance would be shared across classes

  names.each do |name|
    encrypt_attribute name, key_provider: key_provider, ...
  end
end
```

trails has two bodies that both claim to be it:

- `packages/activerecord/src/encryption/encryptable-record.ts` `encrypts()` —
  parses `...namesAndOptions`, seeds the Set, loops `encryptAttribute.call`.
- `packages/activerecord/src/encryption.ts` `encrypts()` — a second arg-parsing
  loop over the same `encryptAttribute`, wired as `Base.encrypts`.

Surfaced while converging `encrypted_attributes` onto `classAttribute()` in
PR #6887: the `self.encrypted_attributes ||= Set.new` seed
(encryptable_record.rb:50) had to be added to **both** bodies, which is exactly
the failure mode a duplicated port produces — the next writer touches one and
the other silently drifts.

Their signatures also differ: encryptable-record's takes
`(...namesAndOptions: unknown[])` and encryption.ts's takes `(klass, ...args)`.
Rails' is one method with `*names` plus named kwargs.

## Converged shape

One `encrypts` living in `encryption/encryptable-record.ts` (the file matching
`encryptable_record.rb`), with Rails' parameter list — `names` plus the kwargs
`key_provider`, `key`, `deterministic`, `support_unencrypted_data`, `downcase`,
`ignore_case`, `previous`, `compress`, `compressor` and the `**context_properties`
rest — and `Base.encrypts` wired to it directly rather than through a second
body. `encryption.ts`'s copy is deleted.

## Acceptance criteria

- [ ] Exactly one `encrypts` body in the repo, in `encryption/encryptable-record.ts`.
- [ ] `Base.encrypts` routes to it; `encryption.ts`'s duplicate is gone.
- [ ] The `self.encrypted_attributes ||= Set.new` seed (encryptable_record.rb:50)
      exists once, not per copy.
- [ ] Encryption suite green; `pnpm parity:api` / `pnpm parity:test` deltas
      non-negative.
