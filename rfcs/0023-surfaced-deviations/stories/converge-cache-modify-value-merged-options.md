---
title: "modify_value opens with merged_options in FileStore and MemoryStore"
status: draft
updated: 2026-08-11
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activesupport"
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

Surfaced while converging `FileStore#modify_value` for
`naming-burndown-activesupport` (PR #6355).

**Rails** (`activesupport/lib/active_support/cache/file_store.rb:222-226`):

```ruby
def modify_value(name, amount, options)
  options = merged_options(options)
  key     = normalize_key(name, options)
  version = normalize_version(name, options)
  amount  = Integer(amount)
```

**trails** (`packages/activesupport/src/cache/file-store.ts`, `modifyValue`)
drops the `merged_options` line and normalizes against whatever the caller
passed. Today the only callers are `increment`/`decrement`
(file_store.rb:61, 81 — ported), which merge first, so the omission is
invisible; it stops being invisible the moment anything calls `modify_value`
with raw call options, and it is a missing call the RFC 0047 call-set gate
would flag if the body were re-extracted.

`MemoryStore#modify_value` has the same `options = merged_options(options)`
first line (`memory_store.rb:223`) and the same omission in
`packages/activesupport/src/cache/memory-store.ts`.

Note `merged_options` is idempotent for an already-merged hash (cache.rb:861-888
re-merges over `@options`), so restoring the line is behaviour-preserving for
the current callers — this is a fidelity fix, not a bug fix.

## Acceptance criteria

1. `FileStore#modifyValue` and `MemoryStore#modifyValue` both open with
   `options = this.mergedOptions(options)`, matching file_store.rb:223 and
   memory_store.rb:223.
2. No behaviour change for the existing `increment`/`decrement` callers —
   `pnpm vitest run packages/activesupport/src/cache` green.
3. `pnpm parity:api:calls` shows no new rows (and retires any
   `call-mismatches-exclude` row that covered the omitted `merged_options`
   call, by hand — only-shrink, no reseed).
