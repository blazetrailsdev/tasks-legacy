---
title: "to_hash and slice! drop Rails' default/default_proc copy (plain object has no default seat)"
status: draft
updated: 2026-08-17
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activesupport"
deps: []
deps-rfc: []
est-loc: 180
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced by PR #6626, which gave `HashWithIndifferentAccess` a real
`default` / `default_proc` seat. Everywhere Rails' Hash carries one and trails
uses a plain JS object, the seat is missing and the corresponding Rails call is
dropped:

- `to_hash` (`vendor/rails/activesupport/lib/active_support/hash_with_indifferent_access.rb:376-381`)
  calls `set_defaults(copy)` on the plain `Hash` it returns. trails'
  `toHash()` returns `Record<string, unknown>`, so the defaults are silently
  lost — the port's `setDefaults` is reached only from `dup`.
- `Hash#slice!` (`core_ext/hash/slice.rb:13-14`) does
  `hash.default = default` / `hash.default_proc = default_proc if default_proc`.
  `packages/activesupport/src/core-ext/hash/slice.ts` carries
  `@missingRailsCall default` for exactly this.
- `test_argless_default_with_existing_nil_key`
  (`vendor/rails/activesupport/test/hash_with_indifferent_access_test.rb:700-704`)
  and `test_default_with_argument` (:706-710) both build their source as
  `Hash.new(:default)` / `Hash.new { 5 }` — a _plain_ Ruby Hash with a default.
  The trails ports substitute the HWIA constructor and say so at the call site.

## Converged shape

Decide where a plain-object default seat lives. The two candidates:

1. Route the affected `core_ext/hash` members through
   `HashWithIndifferentAccess` (or whatever Hash-subclass seat RFC 0098's
   `rebuild-ordered-options-on-a-hash-subclass` settled on), so `to_hash`'s and
   `slice!`'s targets have somewhere to put a default.
2. Accept the plain-object arm as language-forced and ratify it — which the
   CONVERGE-NEVER-RATIFY rule only permits after (1) is shown impossible.

Prefer (1): the seat already exists on the Hash subclass, and it would let
`to_hash` call `set_defaults` and `slice!` copy `default`/`default_proc` at the
Rails lines, retiring one `@missingRailsCall` and restoring the two Rails test
setups verbatim.

## Acceptance criteria

- [ ] `toHash()` calls `setDefaults` at hash_with_indifferent_access.rb:379,
      or the blocker is documented with the specific TS limitation.
- [ ] `core-ext/hash/slice.ts`'s `@missingRailsCall default` is deleted.
- [ ] The two tests above build their source with a real defaulted hash.
