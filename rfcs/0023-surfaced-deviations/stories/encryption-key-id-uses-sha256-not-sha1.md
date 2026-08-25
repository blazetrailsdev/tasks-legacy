---
title: "encryption-key-id-uses-sha256-not-sha1"
status: draft
updated: 2026-08-17
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

# `Encryption::Key#id` uses SHA-256/8 chars where Rails uses SHA-1/4

## Context

`vendor/rails/activerecord/lib/active_record/encryption/key.rb:24-26`:

```ruby
def id
  Digest::SHA1.hexdigest(secret).first(4)
end
```

trails (`packages/activerecord/src/encryption/key.ts:20-22`) computes
`createHash("sha256").update(this.secret).digest("hex").slice(0, 8)` — a
different digest AND a different length, so every key id trails writes into a
message's public tags differs from the one Rails would write for the same
secret. Nothing in the port is Rails-recognisable here beyond the method name.

Surfaced while converging the RFC 0108 accessor call-set rows (the row is
`id | first`, the `String#first(4)` positional idiom); the digest divergence
sits on the same line but is a separate defect, so it was filed rather than
folded into that PR.

## Acceptance criteria

- [ ] `Key#id` is `sha1` hex digest sliced to 4 characters, matching
      `key.rb:25`.
- [ ] Any encryption test or fixture that pinned the 8-character SHA-256 id is
      updated to the Rails value (check `key-provider`, `message`,
      `envelope-encryption-key-provider` specs).
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.
