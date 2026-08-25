---
title: "Port ActiveSupport::Multibyte::Chars and String#mb_chars"
status: draft
updated: 2026-08-25
rfc: "0023-surfaced-deviations"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 250
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`ActiveSupport::Multibyte::Chars` is unported. Its one live caller in the AR
port is `generate_index_name`:

```ruby
# activerecord/lib/active_record/connection_adapters/abstract/schema_statements.rb:1606-1609
short_limit = max_index_name_size - hashed_identifier.bytesize
short_name = name.mb_chars.limit(short_limit).to_s
```

`Chars#limit` is `truncate_bytes(limit, omission: nil)`
(`activesupport/lib/active_support/multibyte/chars.rb:118-120`).

PR #7009 fixed the _behaviour_ — `generateIndexName` had been truncating by JS
UTF-16 length, so a multi-byte table or column name produced a different index
name than Rails on the same schema — by calling ActiveSupport's `truncateBytes`
directly, and left a CONVERGEABLE `@missingRailsCall limit` receipt because
there is no `mb_chars` receiver to call. The remaining work is the receiver.

`String#mb_chars` itself is `activesupport/lib/active_support/core_ext/string/multibyte.rb:1-38`.
Note that only test-local helpers exist today
(`packages/activesupport/src/multibyte-chars.test.ts`,
`multibyte-proxy.test.ts`), so the tests are already written against a shape
that does not exist in `src/` — enrolling them is part of the win here.

## Acceptance criteria

- [ ] `ActiveSupport::Multibyte::Chars` is ported at
      `packages/activesupport/src/multibyte/chars.ts`, with `String#mbChars`
      in `core-ext/string/multibyte.ts`, both at the Rails names.
- [ ] `Chars#limit` delegates to `truncateBytes(limit, { omission: null })`,
      as `chars.rb:119` does.
- [ ] `generateIndexName` calls it at Rails' call site and drops its
      `@missingRailsCall limit` receipt; the existing byte-truncation
      behaviour is unchanged.
- [ ] The existing multibyte test files drive the ported class rather than
      file-local helpers.
- [ ] `pnpm parity:api:calls` / `:args` green; no `--write`, no reseed.
